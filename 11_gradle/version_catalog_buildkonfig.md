# Version catalog & BuildKonfig

## 1. Qué es

Dos mecanismos distintos de configuración Gradle que suelen confundirse porque ambos "centralizan valores", pero resuelven problemas diferentes:

- **Version catalog (`libs.versions.toml`):** un archivo TOML que centraliza en un solo lugar las versiones y coordenadas de todas las dependencias del proyecto (librerías, plugins), generando accesos type-safe (`libs.ktor.client.core`) usables desde cualquier `build.gradle.kts`.
- **BuildKonfig:** un plugin de Gradle que genera código Kotlin en tiempo de compilación a partir de variables de entorno o valores definidos en el build script — pensado específicamente para manejar API keys, Base URLs, y flags que no deberían estar hardcodeados ni commiteados al repo.

## 2. El problema que resuelve

Sin version catalog: cada `build.gradle.kts` de cada módulo declara sus dependencias con la versión escrita a mano (`implementation("io.ktor:ktor-client-core:2.3.7")`). En un proyecto con varios módulos, la misma librería termina con versiones ligeramente distintas entre módulos porque alguien actualizó en un lugar y se olvidó en otro — un problema real de consistencia que además complica actualizar (hay que buscar y reemplazar la versión en N archivos).

Sin BuildKonfig (o algo equivalente): las API keys y Base URLs terminan hardcodeadas directo en el código Kotlin (`const val API_KEY = "sk-abc123"`), lo que significa que quedan commiteadas al repo — visibles para cualquiera con acceso al historial de Git, incluyendo si el repo alguna vez se hace público o se comparte. Además, no hay forma limpia de tener una key distinta para debug vs release, o para desarrollo local vs CI.

## 3. Ejemplo mínimo comentado

```toml
# gradle/libs.versions.toml
[versions]
ktor = "2.3.7"
koin = "3.5.3"
kotlin = "1.9.22"

[libraries]
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
koin-core = { module = "io.insert-koin:koin-core", version.ref = "koin" }

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
```

```kotlin
// feature/players/build.gradle.kts — acceso type-safe generado automáticamente
plugins {
    alias(libs.plugins.kotlin.multiplatform)
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.koin.core)
        }
    }
}
// si mañana hay que actualizar Ktor, se cambia UNA vez en el .toml
// y todos los módulos que usan libs.ktor.client.core quedan actualizados
```

```kotlin
// build.gradle.kts (módulo shared) — configuración de BuildKonfig
buildkonfig {
    packageName = "com.timbax.shared"

    defaultConfigs {
        buildConfigField(
            FieldSpec.Type.STRING,
            "API_BASE_URL",
            System.getenv("API_BASE_URL") ?: "https://api-dev.timbax.com"
        )
    }
}
```

```kotlin
// código Kotlin compartido, consumiendo el valor generado
// BuildKonfig genera una clase BuildKonfig con el campo ya tipado
val client = HttpClient {
    defaultRequest {
        url(BuildKonfig.API_BASE_URL)
    }
}
```

## 4. Matriz de criterio

**Version catalog — usar siempre, sin excepción real:**
- Es la forma recomendada por Google/JetBrains desde hace varias versiones de Gradle, incluso en proyectos de un solo módulo. No modulariza nada por sí solo, pero previene inconsistencias de versión desde el día uno del proyecto.
- Cuando NO tiene sentido: prácticamente nunca — el único caso sería un prototipo descartable de una sola sesión donde ni vale la pena crear el archivo `.toml`.

**BuildKonfig — usar cuando:**
- Necesitás manejar API keys, Base URLs, o cualquier secreto/config que varíe entre debug/release o entre desarrollador/CI.
- Necesitás que ese valor esté disponible como código Kotlin tipado en `commonMain` (no solo en el `build.gradle.kts`, sino consumible desde tu `HttpClient` compartido).

**NO usar BuildKonfig cuando:**
- El valor no es sensible y es genuinamente el mismo en todos los ambientes (por ejemplo, una constante de negocio como `MAX_PLAYERS = 4`) — eso va directo como `const val` en Kotlin, meterlo en BuildKonfig es sobre-ingeniería para algo que no necesita variar.
- Ya estás en un contexto puramente Android sin necesidad de compartir el valor a iOS/Desktop — ahí `BuildConfig` nativo de Android (generado por AGP) alcanza sin sumar una dependencia extra.

**Trade-off real de BuildKonfig:** agrega una dependencia y un paso de configuración extra al build, pero es exactamente lo que hace posible que un secreto nunca toque el código fuente ni el control de versiones — el costo de configurarlo una vez es mínimo comparado con el riesgo de una key filtrada.

## 5. Caso trampa

Te preguntan: *"Ya configuré BuildKonfig para leer `API_KEY` desde una variable de entorno con `System.getenv("API_KEY")`. ¿Ya está resuelto el problema de seguridad?"*

La respuesta que parece obvia es "sí, ya no está hardcodeada en el código, problema resuelto". Es una respuesta incompleta y peligrosa: `System.getenv("API_KEY")` en el `build.gradle.kts` solo mueve el problema de *dónde vive el valor*, pero no resuelve *quién puede llegar a leerlo en runtime*. BuildKonfig termina generando una clase Kotlin (`BuildKonfig.API_KEY`) con el valor embebido como constante en el binario compilado — cualquiera con el APK/IPA final puede decompilarlo y extraer esa key igual, sea de debug o de release. BuildKonfig resuelve el problema de **no commitear la key al repo Git** (que es real y valioso), pero no resuelve el problema de **exponer la key en el binario distribuido**. Para secretos verdaderamente sensibles (keys que dan acceso privilegiado, no solo una key pública de un SDK de terceros), la protección real pasa por otro lado: no embeber la key en el cliente en absoluto, y hacer esas llamadas sensibles a través de un backend propio que las intermedia.

## 6. Conexión con Timbax

Timbax usa Firebase vía GitLive SDK, lo que típicamente implica configuración sensible (el `google-services.json`/`GoogleService-Info.plist`, y potencialmente Base URLs de algún backend propio si Timbax lo tuviera). El version catalog aplica directo hoy: centralizar las versiones de Ktor, Koin, SQLDelight, Compose Multiplatform en un único `libs.versions.toml` es lo que ya se está usando (o debería estarse usando) para evitar que, al actualizar Koin por ejemplo, quede una versión vieja olvidada en algún lado. BuildKonfig entraría en juego el día que Timbax necesite diferenciar un ambiente de desarrollo de uno de producción (por ejemplo, un backend propio de sincronización de partidas) — hoy, si toda la config sensible pasa por los archivos de configuración estándar de Firebase (que ya tienen su propio mecanismo de protección), BuildKonfig todavía no es imprescindible, pero es la herramienta correcta apenas aparezca una URL o key propia del proyecto que deba variar por ambiente.