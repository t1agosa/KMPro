# Remote (Retrofit)

## 1. Qué es

Retrofit es el cliente HTTP type-safe de Square para Android/JVM — la librería de facto para consumir APIs REST en un proyecto Android nativo (no KMP). A diferencia de Ktor, que expone funciones imperativas (`client.get(url)`), Retrofit convierte una **interfaz Kotlin anotada** en una implementación real: cada método describe un endpoint (`@GET`, `@POST`, `@Body`, `@Path`, `@Query`) y Retrofit arma el request HTTP a partir de esas anotaciones. Corre siempre sobre OkHttp como motor de transporte (no es opcional, es dependencia directa), y necesita un `Converter` (Gson, Moshi, o `kotlinx.serialization` vía `converter-kotlinx-serialization`) para transformar el JSON en los DTOs. Desde la versión 2.6.0 soporta funciones `suspend` de forma nativa, y la versión mayor actual es **3.0.0** (mayo 2025), que mantiene compatibilidad binaria con la línea 2.x y trae una versión de OkHttp escrita en Kotlin como dependencia transitiva.

## 2. El problema que resuelve

Sin Retrofit, consumir una API REST en Android nativo significa armar cada `Request` de OkHttp a mano (URL, método, headers, body), ejecutar la call, y parsear la `Response` al DTO correspondiente — boilerplate repetido por cada endpoint. Retrofit resuelve esto con un enfoque declarativo: describís el endpoint como firma de método + anotaciones, y el "cómo" armar y ejecutar el request queda completamente oculto. Es el mismo problema que resuelve Ktor en KMP, pero Retrofit lo resuelve solo para JVM/Android, con una API con más de una década en producción y el ecosistema de converters/adapters más grande del mundo Android — y es, en la práctica, la palabra clave que un reclutador de Android nativo espera ver en un CV.

## 3. Ejemplo mínimo comentado

```kotlin
// Interfaz: describe el contrato, sin implementación — Retrofit la genera en runtime
interface PlayerApi {

    @GET("players")
    suspend fun fetchPlayers(): List<PlayerDto>

    @POST("players")
    suspend fun savePlayer(@Body dto: PlayerDto): PlayerDto

    @GET("players/{id}")
    suspend fun fetchPlayer(@Path("id") id: String): PlayerDto
}
```

```kotlin
// Construcción: se arma una sola vez, típicamente como singleton en el módulo de DI
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.timbax.com/")
    .client(okHttpClient) // OkHttpClient configurado con interceptors, timeouts, etc.
    .addConverterFactory(GsonConverterFactory.create()) // o Moshi / kotlinx-serialization
    .build()

val api: PlayerApi = retrofit.create(PlayerApi::class.java)
```

`retrofit.create(PlayerApi::class.java)` no genera una clase nueva en tiempo de compilación. Retrofit crea un `Proxy` dinámico (`java.lang.reflect.Proxy`) que intercepta cada llamada al método, lee sus anotaciones vía reflection recién cuando se invoca por primera vez, y arma el request de OkHttp correspondiente en ese momento (con el resultado cacheado para las siguientes llamadas). No hay ningún `PlayerApi_Impl` generado por un annotation processor, a diferencia de lo que sucede con Room o Dagger.

## 4. Matriz de criterio

**Retrofit en un proyecto 100% Android nativo (sin KMP):**
- Usar cuando: el proyecto no tiene, ni va a tener a mediano plazo, target iOS/Desktop/Web. Es la opción con más ecosistema, más gente que la reconoce en un code review, y la que filtran los reclutadores por keyword — Retrofit aparece en la enorme mayoría de las búsquedas de "Android Developer" en Argentina/LATAM, incluso en casos donde Ktor sería técnicamente equivalente.
- NO usar cuando: existe aunque sea un plan concreto de compartir la capa de red con iOS. Ahí conviene arrancar directo con Ktor, porque migrar de Retrofit a Ktor después implica reescribir toda la capa `remote` y sus tests.
- Trade-off: madurez y reconocimiento de mercado, a cambio de quedar atado a JVM sin ninguna portabilidad a Kotlin/Native.

**Ktor (lo que usa Timbax) vs. Retrofit — no es "cuál es mejor", es "en qué corre":**
- Usar Ktor cuando: el target es KMP, aunque sea "por las dudas" — como Timbax, que arrancó Android-only pero se armó con Ktor desde el día uno pensando en un eventual build de iOS.
- Usar Retrofit cuando: el target es Android/JVM exclusivamente y no hay intención real de portar a otra plataforma.
- Trade-off real: no es de calidad técnica (ambas son librerías HTTP maduras, con soporte de coroutines de primera clase), es de superficie de compatibilidad — `java.lang.reflect.Proxy`, el mecanismo que usa Retrofit por debajo, no existe en Kotlin/Native, así que ni con el mejor esfuerzo Retrofit va a compilar para iOS.

**Converter: Gson vs. Moshi vs. `kotlinx.serialization`:**
- Usar `converter-kotlinx-serialization` cuando: el proyecto ya usa `kotlinx.serialization` en otro lado, o simplemente para no sumar una segunda librería de serialización — mantiene los mismos DTOs `@Serializable` que se usarían con Ktor.
- Usar Moshi cuando: es Android puro y se prioriza performance de parsing + soporte idiomático de Kotlin (data classes, nulabilidad) sin configuración extra — hoy es la recomendación no oficial más común por sobre Gson en proyectos nuevos.
- Usar Gson cuando: es un proyecto legacy que ya lo tiene instalado hace años. No se recomienda para arrancar un proyecto de cero hoy.
- Trade-off: Gson es el más laxo (deserializa casi cualquier cosa silenciosamente, lo cual puede esconder bugs), Moshi es más estricto y rápido, `kotlinx.serialization` es la opción con mejor integración si el código en algún punto tiene que convivir con Ktor.

**Suspend fun devolviendo el DTO directo vs. devolviendo `Response<T>`:**
- Usar el tipo directo (`suspend fun fetchPlayers(): List<PlayerDto>`) cuando: es el caso general — priorizás simplicidad y manejás los errores vía `try/catch` (ver Caso trampa abajo).
- Usar `Response<T>` cuando: necesitás inspeccionar headers de la respuesta, o distinguir sin pagar el costo de lanzar una excepción entre distintos status codes que en tu dominio no son realmente "errores" (ej: un 404 que significa "todavía no hay datos").
- Trade-off: `Response<T>` da más control sin el costo de construir una excepción para casos esperados, pero rompe la fluidez de "devolver el DTO directo" y obliga a chequear `response.isSuccessful()` a mano en cada método.

## 5. Caso trampa

Un `UseCase` que atrapa el error de una llamada Retrofit con un `catch (e: Exception)` genérico, tratando "no hay internet" y "el servidor respondió con un error" como si fueran la misma cosa:

```kotlin
// ❌ trampa: pierde la distinción entre error de red y error HTTP
suspend fun fetchPlayers(): Result<List<Player>> {
    return try {
        Result.Success(api.fetchPlayers().map { it.toDomain() })
    } catch (e: Exception) {
        Result.Error(e) // "no tenés internet" y "el server tiró un 500" llegan igual acá
    }
}
```

Cuando una función `suspend` de Retrofit devuelve el DTO directamente (sin envolverlo en `Response<T>`), tira dos tipos de excepción bien distintos, no una genérica: `retrofit2.HttpException` cuando el servidor efectivamente respondió pero con un status 4xx/5xx (podés leer `e.code()` y `e.response()?.errorBody()`), e `IOException` (típicamente `SocketTimeoutException` o `UnknownHostException`) cuando la request nunca llegó a completarse — sin conexión, timeout, DNS caído. Tratarlas igual con un `catch (e: Exception)` funciona en el sentido de que la app no crashea, pero le saca a la UI la posibilidad real de mostrar "revisá tu conexión" en vez de "el servidor tuvo un problema", que son mensajes con causas y soluciones completamente distintas para quien juega una partida de Timbax en la mitad de una ronda sin señal.

```kotlin
// ✅ distinguiendo los dos casos, igual que se documentó para Ktor en result_pattern.md
suspend fun fetchPlayers(): Result<List<Player>> {
    return try {
        Result.Success(api.fetchPlayers().map { it.toDomain() })
    } catch (e: HttpException) {
        Result.Error(ServerException(e.code()))
    } catch (e: IOException) {
        Result.Error(NoConnectionException())
    }
}
```

La señal de alarma: cualquier `catch (e: Exception)` envolviendo una llamada Retrofit sin un `catch` previo más específico para `HttpException` es candidato a este error — no es incorrecto todas las veces (a veces de verdad da lo mismo el motivo), pero hay que decidirlo a propósito, no por default.

## 6. Conexión con Timbax

Timbax no usa Retrofit — usa Ktor, documentado en `remote_ktor.md`, precisamente porque se armó pensando en no cerrarse la puerta a iOS. Este archivo existe como contraste, no como arquitectura real del proyecto: si en algún escenario hipotético Timbax se congelara como Android-only (una decisión de negocio válida, no técnica), el swap de Ktor por Retrofit solo tocaría el sub-paquete `remote` de `data` — la interfaz `PlayerApi` pasaría de exponer `HttpClient` a exponer una interfaz anotada, el `Converter` reemplazaría a `ContentNegotiation`, pero el resto de la cadena (`PlayerDto` → mapper `toDomain()` → `PlayerRepositoryImpl` → `domain` → `presentation`) queda exactamente igual. Es la prueba concreta de por qué Clean Architecture separa `data` del resto: la capa de red es un detalle de implementación intercambiable, y ni `domain` ni `presentation` se enteran de si por debajo corre Ktor o Retrofit.