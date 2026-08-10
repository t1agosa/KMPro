# Ktor Client Engines por Plataforma

## 1. Qué es

Los **engines** de Ktor Client son las implementaciones concretas que ejecutan efectivamente una request HTTP en cada plataforma, por debajo de la API común (`HttpClient`, `get {}`, `post {}`) que ya vimos en `remote_ktor.md`. Ktor no reimplementa HTTP desde cero en Kotlin puro para todos los casos — define el contrato común y delega la ejecución real a un engine específico: **OkHttp** en Android/JVM, **Darwin** en iOS/macOS (envuelve `NSURLSession`), **CIO** (Coroutine-based I/O, 100% Kotlin puro, típico en Desktop/Backend), y **Js** para targets Web (delega en `fetch` del navegador).

```kotlin
// androidMain
implementation("io.ktor:ktor-client-okhttp:...")

// iosMain
implementation("io.ktor:ktor-client-darwin:...")

// desktopMain
implementation("io.ktor:ktor-client-cio:...")
```

## 2. El problema que resuelve

HTTP no se implementa de la misma forma en cada sistema operativo: Android tiene su propio stack de networking maduro (OkHttp), iOS tiene el suyo nativo (`NSURLSession`, con integración profunda al sistema — políticas de red en background, certificado del dispositivo, etc.), y la JVM de Desktop no tiene ninguno "oficial" del sistema operativo al que atarse. Si Ktor forzara un único motor HTTP en Kotlin puro para todas las plataformas, perdería toda la madurez, optimización y features específicas (connection pooling, interceptors, caché HTTP, certificate pinning) que cada plataforma ya resolvió mejor de forma nativa. El sistema de engines resuelve esto: cada plataforma usa lo mejor disponible ahí, sin que el código de `commonMain` tenga que saber cuál es.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — solo la API común, sin ningún engine específico
dependencies {
    implementation("io.ktor:ktor-client-core:$ktorVersion")
    implementation("io.ktor:ktor-client-content-negotiation:$ktorVersion")
}

// androidMain — engine OkHttp
dependencies {
    implementation("io.ktor:ktor-client-okhttp:$ktorVersion")
}

// iosMain — engine Darwin
dependencies {
    implementation("io.ktor:ktor-client-darwin:$ktorVersion")
}

// creación del cliente, ya vista en remote_ktor.md — recibe el engine ya resuelto
fun createHttpClient(engine: HttpClientEngine): HttpClient = HttpClient(engine) {
    install(ContentNegotiation) { json() }
}

// androidMain, provisto vía Koin
single { createHttpClient(OkHttp.create()) }

// iosMain, provisto vía Koin
single { createHttpClient(Darwin.create()) }
```

El detalle clave: el engine se declara como dependencia en el sourceSet de cada plataforma (`androidMain`, `iosMain`), nunca en `commonMain`. En muchos proyectos ni siquiera hace falta pasar el engine explícitamente — `HttpClient()` sin argumentos deja que Ktor detecte automáticamente cuál está disponible en el classpath de cada plataforma.

## 4. Matriz de criterio

**OkHttp (Android/JVM):**
- Usar cuando: el target es Android — es el engine por default, con connection pooling, interceptors y caché HTTP maduros, y es el estándar que cualquier dev Android ya conoce.
- NO usar cuando: no aplica en iOS ni en targets no-JVM — es exclusivo de donde corre la JVM.
- Trade-off: ninguno relevante — es la opción sin fricción en Android, no hay razón real para evitarlo ahí.

**Darwin (iOS/macOS):**
- Usar cuando: el target es iOS o macOS — envuelve `NSURLSession`, respetando configuración de red del sistema y políticas de background que un engine puro-Kotlin no podría replicar sin reimplementar media capa de iOS.
- NO usar cuando: no aplica fuera de plataformas Apple.
- Trade-off: ninguno relevante — es, igual que OkHttp en Android, la opción nativa esperable para su plataforma.

**CIO (Coroutine-based I/O):**
- Usar cuando: el target es Desktop (JVM) sin necesidad de features nativas específicas, o del lado servidor (Ktor Server también usa CIO). Es la opción "portable" cuando no hay una alternativa nativa mejor disponible.
- NO usar cuando: hay un engine nativo más maduro disponible para esa plataforma (OkHttp en Android, Darwin en iOS) — CIO existe como fallback multiplataforma, no como reemplazo universal de los engines nativos.
- Trade-off: 100% Kotlin puro, sin dependencias nativas, pero menos optimizado en features específicas de plataforma (caché HTTP avanzada, certificate pinning) que OkHttp o Darwin ya resuelven mejor en su terreno.

**Js (Wasm/JS):**
- Usar cuando: el target es Web — delega directamente en las APIs de `fetch` del navegador, la única opción real disponible ahí.
- NO usar cuando: no aplica fuera de targets Web.
- Trade-off: ninguno relevante — es la única alternativa viable para ese target, no una elección entre varias.

## 5. Caso trampa

Asumir que, como CIO es "100% Kotlin puro y multiplataforma en sí mismo", conviene usarlo en todas las plataformas para simplificar la configuración:

```kotlin
// ❌ trampa: usar CIO en todos los sourceSets "para no tener que pensar en engines por plataforma"
dependencies {
    implementation("io.ktor:ktor-client-cio:$ktorVersion") // en androidMain, iosMain y desktopMain
}
```

Compila y las requests funcionan en desarrollo. El problema aparece en producción: en Android, se pierde el manejo maduro de caché HTTP e interceptors que trae OkHttp (usado también, indirectamente, por buena parte del ecosistema Android que el resto del proyecto ya asume disponible). En iOS es más grave todavía — CIO no está pensado como reemplazo de Darwin ahí, y se pierde la integración con la configuración de red nativa del sistema (políticas de background, comportamiento ante cambios de conectividad) que sí tiene `NSURLSession` por debajo de Darwin.

La pregunta que expone la trampa: "¿por qué no usar siempre CIO ya que es 100% Kotlin puro?" — la respuesta correcta no es "porque no compila", compila perfecto; es que CIO es la opción portable para cuando no hay una alternativa nativa mejor, no la opción por defecto que conviene elegir en todos lados solo por uniformidad.

## 6. Conexión

En Timbax, `PlayerApi` (documentado en `remote_ktor.md`) usa el mismo código de `commonMain` sin importarle qué engine hay detrás — Koin resuelve esa diferencia inyectando `OkHttp.create()` en `androidMain` y `Darwin.create()` en `iosMain`, ambos como implementación del mismo `HttpClient` que el resto de la capa `data` consume. Este mismo patrón — API común en `commonMain`, implementación real resuelta por plataforma — es conceptualmente el mismo mecanismo que `expect/actual` (documentado en `05_platform`), aunque acá no se use esa sintaxis explícita: Ktor ya resuelve el "puente con capacidades nativas" a su manera, vía engines intercambiables por configuración de Gradle en vez de declaraciones `expect fun`.