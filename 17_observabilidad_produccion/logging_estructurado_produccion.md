# Logging estructurado en producción

## 1. Qué es

Logging estructurado es registrar eventos de la app como datos con forma (nivel de severidad, mensaje, tags, pares clave-valor de contexto), no como strings sueltos concatenados a mano. En KMP el estándar de facto es **Kermit** (Touchlab): expone una API común de logging en `commonMain` que en runtime escribe a distintos **outputs por plataforma** (Logcat en Android, `os_log`/NSLog en iOS, consola en Desktop/Web) a través de un sistema de **writers** (o *sinks*) configurables y componibles — podés tener varios writers activos a la vez, cada uno con su propio nivel mínimo de severidad.

## 2. El problema que resuelve

`println()` o el logger nativo de cada plataforma por separado tiene tres problemas en producción:

- **No es multiplataforma**: tenés que escribir y mantener lógica de logging distinta en cada `platformMain`.
- **No es estructurado**: un string tipo `"Error guardando score: $e"` es difícil de filtrar, buscar o correlacionar cuando después vive mezclado con miles de líneas en un backend de crash reporting.
- **No distingue dev de producción**: sin un mecanismo de niveles y writers, terminás con logs de debug ensuciando producción, o directamente sin ningún log llegando a ningún lado cuando el usuario real tiene un problema que vos no podés reproducir.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — configuración base de Kermit
import co.touchlab.kermit.Logger
import co.touchlab.kermit.Severity
import co.touchlab.kermit.StaticConfig
import co.touchlab.kermit.platformLogWriter

val logger = Logger(
    config = StaticConfig(
        minSeverity = Severity.Debug, // en debug build, todo pasa
        logWriterList = listOf(platformLogWriter()) // Logcat / os_log según plataforma
    ),
    tag = "Timbax"
)

// uso en cualquier capa de commonMain
logger.i { "Score guardado para player=${player.id}" }
logger.e(throwable) { "Fallo al guardar score" }
```

```kotlin
// androidMain — sumamos un writer de producción (Crashlytics vía GitLive)
import co.touchlab.kermit.Severity

val crashlyticsWriter = FirebaseCrashlyticsWriter(minSeverity = Severity.Warn)

val productionLogger = Logger(
    config = StaticConfig(
        minSeverity = Severity.Debug,
        logWriterList = listOf(platformLogWriter(), crashlyticsWriter)
        // platformLogWriter: siempre, para ver todo en local
        // crashlyticsWriter: solo Warn+ para no inundar el backend
    )
)
```

## 4. Matriz de criterio

| Escenario | Usar | NO usar | Trade-off |
|---|---|---|---|
| Debug local, cualquier capa | `logger.d` / `logger.v` con writer de consola | Enviar a Crashlytics | Ruido innecesario en el backend de producción |
| Excepción real en producción | `logger.e(throwable) { }` con writer que reporta (Crashlytics/Sentry/Bugsnag) | Solo loguear localmente | Sin writer remoto, el error nunca llega a vos |
| Contexto de negocio (playerId, gameId) | Pares clave-valor estructurados, no interpolación libre en el string | `"Error para $player"` sin estructura | El string libre no es filtrable ni agregable en el backend |
| Multiplataforma (Android + iOS + Desktop) | Kermit (`platformLogWriter()` resuelve el output nativo por plataforma) | Lógica de logging duplicada en cada `platformMain` | Ninguno real — es la solución estándar KMP |
| Nivel de severidad en release build | `minSeverity = Warn` o superior para writers remotos | Mandar `Debug`/`Verbose` a producción | Satura el backend y encarece el plan del proveedor |

## 5. Caso trampa

**"Ya logueo la excepción con `logger.e(throwable)`, con eso alcanza para debuggear un crash reportado por un usuario en producción."**

Falso si el log no viaja con contexto. Un `logger.e` sin datos asociados (qué pantalla, qué evento disparó la acción, qué estado tenía el ViewModel) te da el stack trace pero no el *por qué* — en local podés reproducir con el debugger, en producción no. La respuesta correcta es adjuntar contexto estructurado antes de que ocurra el error (custom keys en Crashlytics, breadcrumbs en Sentry) para que cuando el error explote, el reporte ya venga con la foto completa del estado, no solo el stack.

Trampa relacionada, específica de iOS/Kotlin Native: un crash **fatal** en iOS termina generando **dos reportes separados** en el backend (uno del sistema, con `konan::abort()`, y uno no-fatal con el stack de Kotlin) — no es un bug de tu logging, es una limitación conocida de cómo Kotlin/Native propaga excepciones no capturadas hacia el runtime de iOS. Si ves duplicados, no asumas que tu writer está mal configurado.

## 6. Conexión con Timbax

Timbax ya usa Firebase vía GitLive SDK — el camino natural es Kermit + un writer que reporte a Firebase Crashlytics (también vía GitLive) con `minSeverity = Warn`, dejando `platformLogWriter()` corriendo siempre en paralelo para debug local. Un buen primer punto de instrumentación real: envolver `SaveScoreUseCase` con `logger.e(throwable) { "..." }` en el catch, con el `playerId` y el `score` como contexto — así un fallo real de guardado en producción llega con los datos mínimos para reproducirlo, sin necesitar que el usuario te lo describa a mano.