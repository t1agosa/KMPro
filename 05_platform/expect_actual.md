# expect_actual.md

## 1. Qué es

`expect/actual` es el mecanismo de Kotlin Multiplatform para declarar que una función, clase, propiedad u objeto existe en `commonMain` con una firma común, pero cada plataforma provee su propia implementación concreta. `expect` es la declaración (sin cuerpo) en código compartido; `actual` es la implementación real en cada sourceSet de plataforma (`androidMain`, `iosMain`, etc).

```kotlin
// commonMain
expect fun getPlatformName(): String
```

```kotlin
// androidMain
actual fun getPlatformName(): String = "Android"

// iosMain
actual fun getPlatformName(): String = "iOS"
```

## 2. El problema que resuelve

Código compartido en `commonMain` no puede llamar directamente APIs nativas de plataforma — no existe una "librería estándar" que abstraiga vibración, biometría, `UIDevice` de iOS o `Settings.Secure` de Android. Sin `expect/actual`, las alternativas serían: (a) escribir la lógica que necesita esa capacidad por completo en cada plataforma por fuera de `commonMain` — duplicando todo lo demás alrededor de esa única línea que difiere —, o (b) inventar algún tipo de reflexión/casting inseguro que Kotlin ni siquiera soporta igual en Kotlin/Native. `expect/actual` resuelve esto dejando que el *resto* de la lógica (que sí es común) viva compartida, y aislando solo el punto de contacto con la plataforma en una firma que el compilador obliga a implementar en todos lados — si te olvidás de un `actual` en algún target, no compila. Es una garantía en tiempo de compilación, no una convención de buenas prácticas.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — la firma común, sin implementación
expect fun getDeviceId(): String
```

```kotlin
// androidMain — implementación específica de Android
actual fun getDeviceId(): String =
    Settings.Secure.getString(context.contentResolver, Settings.Secure.ANDROID_ID)
```

```kotlin
// iosMain — implementación específica de iOS, vía cinterop con UIKit
actual fun getDeviceId(): String =
    platform.UIKit.UIDevice.currentDevice.identifierForVendor?.UUIDString ?: ""
```

El código que *llama* a `getDeviceId()` vive en `commonMain` y no sabe (ni le importa) qué implementación se ejecuta — eso lo resuelve el compilador en cada target.

## 4. Matriz de criterio

**Usar cuando:**
- La necesidad es una función simple, sin estado propio, sin dependencias que inyectar (una lectura puntual de una API nativa: nombre de plataforma, ID de dispositivo, un valor de configuración del sistema).
- Es un puente puro hacia una capacidad nativa (hardware, SO), no una decisión de negocio.
- Necesitás que el compilador te obligue a implementarlo en cada plataforma sí o sí (falla en compile-time si te olvidás, no en runtime).

**NO usar cuando:**
- La implementación por plataforma necesita recibir dependencias (contexto de Android, un cliente HTTP configurado, otro objeto). Ahí una interfaz + implementación inyectada por Koin es más flexible — el `single<T> { PlatformImpl(dep1, dep2) }` te permite pasar lo que haga falta, cosa que `actual` no resuelve limpio (vas a terminar necesitando otro `expect/actual` solo para pasar el contexto).
- Necesitás testear con fakes fácilmente — con `expect/actual` no hay una interfaz de por medio para mockear/fakear en tests de domain o presentation; con DI, inyectás un `FakePlatformService` sin problema.
- Se trata de lógica de negocio con ramificaciones (más que "traer un dato nativo") — eso no es platform, es domain/usecase.

Este eje (expect/actual vs interfaz+DI) se profundiza en `expect_actual_vs_interfaz_di.md`, el próximo archivo del package.

**Trade-off real:** `expect/actual` es más liviano (cero indirección, resuelto en compile-time, sin overhead de grafo de DI) pero más rígido (no admite parámetros de construcción variables entre plataformas de forma natural, y te ata a declarar el `actual` en cada target aunque solo lo necesites en uno).

## 5. Caso trampa

Tenés `expect fun vibrate(durationMs: Long)` funcionando bien en Android e iOS. Ahora agregás un target Desktop al proyecto (JVM) porque Timbax va a tener una versión de escritorio. El proyecto **deja de compilar** — no porque el código de Desktop esté mal, sino porque no existe ningún `actual fun vibrate(...)` en `desktopMain`. La reacción instintiva es "bueno, hago un `actual` que no haga nada" (`actual fun vibrate(durationMs: Long) {}`), pero eso es parchar el síntoma: la pregunta real es si `vibrate` debería seguir siendo `expect/actual` en absoluto, dado que ahora hay una plataforma donde el concepto "vibrar" ni siquiera aplica. Ahí conviene evaluar si el llamador debería manejar la ausencia de la capacidad explícitamente (una interfaz con una implementación `NoOpHapticFeedback` para Desktop, inyectada por Koin) en vez de forzar un `actual` vacío silencioso en cada plataforma nueva que agregues — el `expect/actual` vacío esconde la decisión, la interfaz la hace explícita.

## 6. Conexión con arquitectura real

En Timbax, `expect/actual` es candidato natural para algo como compartir el resultado de una partida (share sheet nativo) o detectar si el dispositivo soporta biometría antes de mostrar una opción de bloqueo de app — capacidades puntuales sin estado ni dependencias complejas. Notá que **nunca** debería aparecer un `expect/actual` dentro de `domain` — domain es Kotlin puro sin saber siquiera que existen plataformas; el punto de contacto vive en `platform` (o en `data`, si la capacidad se usa para implementar un repository, como un `actual` que decide qué motor de Ktor usar, ver `ktor_engines_por_plataforma.md`).