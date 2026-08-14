# Cifrado de datos en tránsito

## 1. Qué es

Es la protección de los datos mientras viajan por la red, entre el cliente (tu app) y el servidor — a diferencia del cifrado "en reposo" (datos guardados en disco, DB, DataStore), que es un problema distinto con sus propias herramientas. El mecanismo estándar es TLS (Transport Layer Security, sucesor de SSL): cifra el canal completo para que nadie que intercepte el tráfico pueda leer ni modificar los datos en tránsito.

HTTPS no es más que HTTP corriendo sobre TLS. Cuando tu `HttpClient` de Ktor apunta a una URL `https://`, ya estás usando cifrado en tránsito por default — no es algo que actives a mano, es el comportamiento estándar de cualquier engine (OkHttp, Darwin, CIO) contra un endpoint HTTPS bien configurado.

## 2. El problema que resuelve

Sin TLS, cualquier persona con acceso a la red por la que viaja el tráfico — un Wi-Fi público, un router comprometido, un proxy corporativo mal configurado — puede leer el contenido crudo de cada request y response: tokens de sesión, contraseñas, datos personales del usuario. Esto se llama ataque **man-in-the-middle (MITM)**: un tercero se posiciona entre el cliente y el servidor, interceptando (y potencialmente modificando) el tráfico sin que ninguna de las dos partes lo note.

TLS resuelve esto con cifrado + verificación de identidad: el cliente valida que el certificado del servidor fue emitido por una autoridad certificadora (CA) confiable y corresponde al dominio al que se está conectando, antes de establecer el canal cifrado. El problema real que queda abierto (y que TLS por sí solo no resuelve) es: ¿qué pasa si el atacante logra que tu dispositivo confíe en un certificado falso? Ahí entra **certificate pinning**, un nivel extra de protección.

## 3. Ejemplo mínimo comentado

TLS "gratis" simplemente usando HTTPS — no hace falta código extra:

```kotlin
// commonMain — esto YA viaja cifrado, TLS es transparente
val client = HttpClient {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
}

suspend fun getPlayers(): List<PlayerDto> =
    client.get("https://api.timbax.com/players").body()
```

Certificate pinning explícito (un paso más allá de TLS estándar), configurado por engine ya que cada plataforma expone su propia API nativa para esto:

```kotlin
// androidMain — engine OkHttp
actual fun createHttpClient(): HttpClient = HttpClient(OkHttp) {
    engine {
        config {
            certificatePinner(
                CertificatePinner.Builder()
                    .add("api.timbax.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
                    .build()
            )
        }
    }
}
```

```kotlin
// iosMain — engine Darwin, API propia inspirada en OkHttp
actual fun createHttpClient(): HttpClient = HttpClient(Darwin) {
    engine {
        val builder = CertificatePinner.Builder()
            .add(pattern = "api.timbax.com", pins = pinnedHashes)
        handleChallenge(builder.build())
    }
}
```

## 4. Matriz de criterio

**Usar TLS estándar (sin pinning extra) cuando:**
- Es cualquier app que hable con un backend propio — no es opcional, es la base mínima innegociable. No hay escenario legítimo donde valga la pena mandar datos en claro por HTTP.
- El riesgo de MITM está cubierto razonablemente por la validación estándar de cadena de certificados (CA confiable + dominio correcto).

**Sumar certificate pinning cuando:**
- La app transfiere datos sensibles (pagos, datos personales regulados, credenciales) y el costo de un MITM exitoso es alto.
- Es un contexto de alto riesgo de red no confiable (apps usadas en países con infraestructura de red intervenida, redes corporativas con proxies MITM legítimos pero indeseados para tu app).

**NO usar pinning cuando:**
- El equipo no tiene proceso de rotación de certificados/pines coordinado con releases de la app — un cert que expira sin que el pin se haya actualizado en una versión ya publicada **rompe la app en producción para todos los usuarios en esa versión**, sin posibilidad de fix server-side. Es el trade-off central: pinning es seguridad real, pero es una responsabilidad operativa permanente, no un flag que se prende y se olvida.
- El backend rota certificados frecuentemente (ej: certificados gestionados automáticamente con renovación corta) sin un mecanismo de pines múltiples/backup — mismo riesgo de romper la app.
- Es un proyecto chico o interno donde el costo de mantenimiento no se justifica frente al riesgo real.

**Trade-off real:** TLS estándar es gratis y obligatorio. Pinning agrega una capa de seguridad genuina contra MITM con certificados falsos válidos, pero convierte cada renovación de certificado del backend en un evento coordinado que puede tumbar la app si se gestiona mal.

## 5. Caso trampa

"Ya tenemos HTTPS, así que estamos protegidos contra MITM" — esto es cierto contra un atacante que no puede conseguir un certificado válido para tu dominio. Pero no cubre el escenario donde el atacante logra que el **dispositivo** confíe en un certificado falso (por ejemplo, instalando un certificado raíz malicioso en el dispositivo — típico en ataques dirigidos, o en entornos donde el usuario instaló sin saberlo un perfil de configuración comprometido). En ese caso, TLS estándar valida correctamente la cadena de confianza... contra una cadena de confianza que ya está comprometida. La app confía en el certificado falso porque el sistema operativo también confía en él.

Certificate pinning es justamente la respuesta a esto: en vez de confiar en "cualquier certificado válido según el sistema", el cliente valida contra un certificado o clave pública específica que vos definiste — así que aunque el dispositivo tenga un certificado raíz comprometido instalado, tu app igual rechaza la conexión si no coincide con el pin esperado.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el `HttpClient` de Ktor se construye vía Koin con el engine específico por plataforma (`OkHttp` en `androidMain`, `Darwin` en `iosMain`) — como ya está documentado en `03_data/ktor_engines_por_plataforma.md`. TLS estándar ya está cubierto automáticamente por usar `https://` contra Firebase/backend. Certificate pinning explícito no está implementado hoy porque Timbax no maneja datos financieros ni PII sensible más allá de nombres de jugadores y puntajes — encaja con el criterio de "NO usar pinning" de la sección 4: el costo operativo de mantener pines coordinados con la rotación de certificados de Firebase no se justifica frente al riesgo real del dominio de la app.