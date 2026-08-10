# Remote (Ktor)

## 1. Qué es

Ktor Client es la librería oficial de JetBrains para hacer llamadas HTTP desde Kotlin Multiplatform. Vive en la capa `data`, específicamente en el sub-paquete `remote`, y es la implementación concreta que le da contenido real al `Api` (o `PlayerApi`, `RemoteDataSource`, según cómo se lo nombre) que después consume `RepositoryImpl`. Expone una API común (`HttpClient`, `get {}`, `post {}`) que compila igual en `commonMain`, pero por debajo delega la ejecución real de la request a un motor (engine) específico por plataforma — OkHttp en Android, Darwin en iOS, CIO en Desktop.

```kotlin
class PlayerApi(private val client: HttpClient) {

    suspend fun fetchPlayers(): List<PlayerDto> =
        client.get("https://api.timbax.com/players").body()

    suspend fun savePlayer(dto: PlayerDto): PlayerDto =
        client.post("https://api.timbax.com/players") {
            contentType(ContentType.Application.Json)
            setBody(dto)
        }.body()
}
```

## 2. El problema que resuelve

Sin Ktor, cada plataforma de KMP necesitaría su propio cliente HTTP escrito a mano (o una librería nativa distinta por plataforma), y el código compartido en `commonMain` no tendría forma de hacer una sola llamada de red que compile para Android, iOS y Desktop a la vez — habría que triplicar la lógica de "armar la request, mandarla, parsear la respuesta" en cada `platformMain`. Ktor resuelve esto exponiendo una única API multiplataforma real (no un wrapper que finge serlo): el mismo código `client.get(url)` corre en las tres plataformas, y solo el engine que ejecuta la request por debajo cambia según dónde se compile.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — configuración del cliente, sin acoplarse a ningún engine específico
fun createHttpClient(engine: HttpClientEngine): HttpClient = HttpClient(engine) {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true }) // tolera campos nuevos del backend sin crashear
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 10_000
    }
}

// data/remote — el DataSource que usa el cliente
class PlayerApi(private val client: HttpClient) {

    suspend fun fetchPlayers(): List<PlayerDto> =
        client.get("https://api.timbax.com/players").body()
}

// androidMain — provisto vía Koin con el engine de esa plataforma
single { createHttpClient(OkHttp.create()) }

// iosMain
single { createHttpClient(Darwin.create()) }
```

`ignoreUnknownKeys = true` es la línea más importante de todo el bloque: sin ella, si el backend agrega un campo nuevo al JSON un día antes de que la app se actualice, la deserialización explota en producción. Con esa flag, Kotlinx Serialization simplemente ignora lo que no reconoce.

## 4. Matriz de criterio

**Engine por defecto (dejar que Ktor detecte automáticamente cuál hay en el classpath):**
- Usar cuando: el proyecto es chico/mediano y no necesitás configuración fina por plataforma (certificate pinning custom, interceptors específicos de OkHttp, etc.) — simplemente instanciás `HttpClient()` sin argumentos en cada sourceSet de plataforma.
- NO usar cuando: necesitás controlar explícitamente qué engine se usa (por testing, por requerimientos de seguridad específicos, o para inyectarlo vía Koin de forma testeable con un engine fake).
- Trade-off: menos código de configuración, pero menos control y menos explícito para alguien que lee el proyecto por primera vez.

**`ContentNegotiation` + `kotlinx.serialization` (la combinación estándar KMP):**
- Usar cuando: siempre que el backend hable JSON — es la ruta oficial y la que mejor integra con el resto del ecosistema Kotlin (los mismos DTOs `@Serializable` sirven para parsear la respuesta sin conversión manual).
- NO usar cuando: el backend usa un formato no estándar (XML, protobuf) — ahí Ktor tiene plugins específicos, pero la configuración cambia.
- Trade-off: ninguno relevante en el caso estándar — es la opción sin fricción para JSON en KMP.

**Manejo de errores HTTP: `expectSuccess = true` vs. chequeo manual de `response.status`:**
- Usar cuando `expectSuccess = true` (default en versiones recientes de Ktor): querés que cualquier respuesta 4xx/5xx lance una excepción automáticamente, para capturarla en el `try/catch` del `UseCase` (como ya se documentó en `result_pattern.md`) sin código extra en el `Api`.
- Usar chequeo manual (`if (response.status != HttpStatusCode.OK)`) cuando: necesitás distinguir comportamiento distinto según el código de error específico (ej: un 401 dispara un refresh de token, un 404 no es realmente un "error" sino "no hay datos todavía") — ahí conviene inspeccionar el status antes de dejar que la excepción genérica se propague.
- Trade-off: `expectSuccess = true` es menos código y más consistente con el patrón `Result` ya armado en domain, pero pierde granularidad si necesitás reaccionar distinto según el tipo exacto de error HTTP.

**Retrofit como alternativa (mencionado en la fuente, para contraste):**
- Usar cuando: nunca en un proyecto KMP real — Retrofit es exclusivo de Android/JVM, no corre en iOS ni Desktop nativo.
- NO usar cuando: cualquier proyecto que apunte a más de una plataforma. Se menciona acá solo porque es la referencia que cualquier dev Android ya conoce, y ayuda a entender por qué Ktor es la elección obligada en KMP y no una preferencia estilística.

## 5. Caso trampa

Un `HttpClient` que se crea de nuevo en cada llamada, en vez de vivir como singleton inyectado por Koin:

```kotlin
// ❌ trampa: un HttpClient nuevo por cada request
class PlayerApi {
    suspend fun fetchPlayers(): List<PlayerDto> {
        val client = HttpClient(OkHttp.create()) {
            install(ContentNegotiation) { json() }
        }
        return client.get("https://api.timbax.com/players").body()
        // el client nunca se cierra, y además se descarta connection pooling, keep-alive, etc.
    }
}
```

El código "funciona" — cada llamada de red efectivamente responde. El problema es de performance y de recursos: `HttpClient` está diseñado para vivir como una instancia única durante todo el ciclo de vida de la app (por eso se declara `single {}` en Koin), porque internamente mantiene un pool de conexiones reutilizables. Crear uno nuevo por cada request tira ese pool a la basura en cada llamada, agrega latencia innecesaria (cada request negocia una conexión nueva desde cero), y en plataformas como Android puede generar leaks de sockets si el `HttpClient` nunca se cierra explícitamente con `.close()`.

La señal de alarma a reconocer: si en algún lugar del código aparece `HttpClient(...)` fuera de un módulo de Koin (`single { ... }`) o de una función factory llamada una sola vez al arrancar la app, es casi seguro un caso de este error.

## 6. Conexión

En Timbax, el `HttpClient` se configura una sola vez por plataforma (con su engine correspondiente) y se inyecta vía Koin como `single` hacia `PlayerApi`, que a su vez es una dependencia de `PlayerRepositoryImpl` (ya visto en `repository_impl.md`). El flujo completo queda: `HttpClient` (Ktor, por plataforma) → `PlayerApi` (remote data source) → `PlayerDto` → mapper `toDomain()` (visto en `dto_entity_mappers.md`) → `PlayerRepositoryImpl` decide si ese dato llega a la DB local antes de llegar a la UI. Ktor es, en ese flujo, la puerta de entrada de todo lo que viene de afuera de la app — y como cualquier otra pieza de `data`, el resto de las capas no sabe ni le importa que existe.