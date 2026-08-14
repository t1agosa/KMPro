# BuildKonfig Multi-Ambiente

## 1. Qué es

BuildKonfig es la librería que ya vimos en `todosobreKMP` (sección Gradle) para manejar variables de entorno de forma type-safe en KMP. Este archivo profundiza específicamente en su uso para **múltiples ambientes** (dev/staging/prod): genera un objeto Kotlin (tipo `BuildConfig` de Android, pero multiplataforma) con los valores correspondientes al ambiente activo, accesible desde `commonMain`.

La selección de ambiente se hace con una property de Gradle: `buildkonfig.flavor`, seteada en `gradle.properties` o pasada por línea de comando (`-Pbuildkonfig.flavor=staging`) — **es independiente del flavor de Android Gradle Plugin**, aunque en la práctica se sincronizan a propósito (ver `flavors_y_schemes_por_plataforma.md`).

## 2. El problema que resuelve

Sin esto, la única forma de que el código compartido (`domain`/`data`) sepa contra qué URL pegarle es:
- Hardcodear la URL y cambiarla a mano antes de cada build (frágil, propenso a error humano).
- Duplicar lógica de configuración por plataforma (un `BuildConfig` en Android, un `.xcconfig`/`Info.plist` en iOS), sin una fuente única de verdad ni acceso type-safe desde Kotlin compartido.

BuildKonfig centraliza la definición en un solo lugar (`build.gradle.kts` del módulo compartido) y genera el objeto correspondiente para cada plataforma automáticamente.

## 3. Ejemplo mínimo comentado

```kotlin
// shared/build.gradle.kts
import com.codingfeline.buildkonfig.compiler.FieldSpec.Type.STRING

buildkonfig {
    packageName = "com.timbax.shared"
    objectName = "TimbaxConfig"

    // defaultConfigs es obligatorio: valores base si no hay flavor activo
    defaultConfigs {
        buildConfigField(STRING, "BASE_URL", "https://dev.timbax.com")
    }

    // defaultConfigs("nombre") sobrescribe los valores base cuando
    // buildkonfig.flavor coincide con "nombre"
    defaultConfigs("staging") {
        buildConfigField(STRING, "BASE_URL", "https://staging.timbax.com")
    }

    defaultConfigs("prod") {
        buildConfigField(STRING, "BASE_URL", "https://api.timbax.com")
    }
}
```

```properties
# gradle.properties (o pasado por -P en CI)
buildkonfig.flavor=staging
```

```kotlin
// uso desde código compartido, ya type-safe
class KtorClientProvider {
    fun baseUrl(): String = TimbaxConfig.BASE_URL
}
```

La prioridad de merge cuando hay valores en varios niveles es: `targetConfigs("flavor")` > `targetConfigs` (sin flavor) > `defaultConfigs("flavor")` > `defaultConfigs` (base) — es decir, lo más específico por plataforma+flavor siempre gana.

## 4. Matriz de criterio

| Escenario | Usar BuildKonfig | NO usar / alternativa |
|---|---|---|
| Configuración simple: URLs, flags de feature, nombres por ambiente | Sí, caso de uso central | — |
| Necesitás que el valor esté disponible sin recompilar (cambiar en runtime) | No — BuildKonfig genera valores fijos en compile-time | Remote Config (Firebase) o un endpoint de configuración remota |
| API keys realmente sensibles (no solo config) | Usar BuildKonfig como transporte, pero el valor real nunca se commitea — se inyecta desde `gradle.properties` local (gitignored) o GitHub Secrets en CI | Ver `secretos_gitignore.md` — BuildKonfig no reemplaza la gestión de secretos, solo la expone type-safe |
| Distinción de valores por plataforma dentro del mismo ambiente (ej: distinto `Client ID` en Android vs iOS para el mismo backend) | Sí, con `targetConfigs` | — |
| Proyecto sin necesidad real de múltiples ambientes (solo prod) | Innecesario — un solo `defaultConfigs` alcanza, no hace falta el sistema de flavors de BuildKonfig | — |

## 5. Caso trampa

**"Configuré `defaultConfigs('staging')` pero al compilar sigue usando los valores de dev."**

La trampa: `buildkonfig.flavor` tiene que existir como property de Gradle *antes* de que se evalúe el plugin — si la seteás dinámicamente dentro del mismo `build.gradle.kts` con lógica condicional mal ubicada (por ejemplo, después del bloque `buildkonfig { }`), el plugin ya leyó el valor por default. La forma correcta es fijarla en `gradle.properties`, pasarla por `-P` en el comando de build, o (si viene de un scheme de Xcode / variante de Android) setearla vía `project.extra.set("buildkonfig.flavor", ...)` **antes** de que Gradle evalúe el bloque `buildkonfig { }` — no después.

Otro trampa relacionada: los nombres de flavor en `defaultConfigs("staging")` son strings libres — un typo (`"stagin"`) no da error de compilación, simplemente ese flavor nunca hace match y silenciosamente cae al `defaultConfigs` base. Vale la pena revisar el valor generado (`generateBuildKonfig` deja el archivo generado inspeccionable) después de cambiar el flavor activo, en vez de asumir que compiló bien porque no tiró error.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, `TimbaxConfig.BASE_URL` (o el nombre de objeto que se elija) sería consumido exclusivamente en la capa `data`, específicamente al construir el `HttpClient` de Ktor (ver `remote_ktor.md`) — nunca en `domain` ni en `presentation`. Esto respeta la Dependency Rule: el `UseCase` que pide jugadores no sabe ni le importa si el `BASE_URL` es de staging o de prod, solo el `RepositoryImpl`/factory de Ktor necesita esa información al momento de armar el cliente HTTP.