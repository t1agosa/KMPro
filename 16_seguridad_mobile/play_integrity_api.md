# Play Integrity API

## 1. Qué es

Es la API de Google que le permite a tu app pedirle a los servidores de Google Play un **veredicto firmado** sobre tres preguntas: ¿este binario es el que Google Play reconoce sin modificar (no fue tampereado/repackageado)?, ¿este dispositivo es genuino y certificado (no es un emulador ni tiene root)?, y ¿esta cuenta instaló/pagó la app legítimamente en Google Play? La API ayuda a verificar que las acciones del usuario y los requests al servidor vienen de tu app genuina, instalada por Google Play, corriendo en un dispositivo Android certificado y genuino.

Es **Android-only** — no existe un equivalente directo en iOS vía Play Integrity (Apple tiene su propio mecanismo, App Attest). El flujo siempre involucra tu backend: la app pide el veredicto, pero **quien lo valida es tu servidor**, nunca el cliente solo.

## 2. El problema que resuelve

Sin esto, tu backend no tiene forma confiable de saber si un request realmente viene de tu app corriendo en un dispositivo legítimo, o si viene de: una versión modificada de tu APK (código tampereado, ads removidos, lógica de negocio parcheada), un emulador o farm de emuladores automatizando abuso (bots, fraude de referidos, cheating en un juego), o una cuenta que nunca pagó/instaló legítimamente la app pero igual está pegándole a tu API.

Cuando un usuario realiza una acción, la app pide una evaluación a la Play Integrity API. El servidor de Google Play devuelve una respuesta encriptada con un veredicto de integridad que la app reenvía a tu servidor para verificación. Tu backend usa ese veredicto para decidir qué hace tu app o juego a continuación. Antes existía SafetyNet Attestation para este mismo propósito, pero está deprecado — Play Integrity es su reemplazo oficial.

## 3. Ejemplo mínimo comentado

El flujo real tiene tres partes: pedir el token en el cliente (Android-only, vía `expect/actual`), mandarlo al backend, y que el backend lo valide contra Google.

```kotlin
// commonMain
expect suspend fun requestIntegrityToken(nonce: String): String
```

```kotlin
// androidMain — usa el SDK de Play Integrity
actual suspend fun requestIntegrityToken(nonce: String): String {
    val integrityManager = IntegrityManagerFactory.create(context)
    val request = IntegrityTokenRequest.builder()
        .setNonce(nonce) // nonce generado por TU backend, no por el cliente
        .build()
    return integrityManager.requestIntegrityToken(request).await().token()
}
```

```kotlin
// commonMain — UseCase que orquesta el chequeo antes de una acción sensible
class VerifyDeviceIntegrityUseCase(
    private val repository: IntegrityRepository
) {
    suspend operator fun invoke(): Result<Unit> {
        val nonce = repository.getNonceFromBackend()
        val token = requestIntegrityToken(nonce)
        return repository.verifyTokenWithBackend(token) // backend valida contra Google
    }
}
```

El backend nunca confía en lo que dice el cliente directamente — reenvía el token firmado a los servidores de Google para decodificarlo y confirmar el veredicto.

## 4. Matriz de criterio

**Usar Play Integrity cuando:**
- Hay acciones sensibles del lado del servidor que se pueden abusar: canjear un premio, procesar un pago, subir un score competitivo a un leaderboard, crear cuentas masivamente.
- La app maneja dinero, identidad, o sistemas competitivos — ahí Strong Integrity deja de ser opcional y pasa a ser esperado.
- Ya usás Firebase en el proyecto: Firebase App Check permite recibir un veredicto de integridad de app y dispositivo potenciado por Play Integrity API en dispositivos Android certificados, junto con respuestas de otros proveedores de attestation de plataforma. Esto simplifica mucho la integración si ya tenés el SDK de Firebase (como Timbax vía GitLive).

**NO usar (o usar con enforcement laxo) cuando:**
- Es una app sin backend propio, o sin ninguna acción que valga la pena proteger contra abuso (no hay economía en juego, no hay leaderboard competitivo, no hay compras).
- El enforcement es "todo o nada": en 2026 lo que importa no es solo implementar Play Integrity, sino elegir el nivel correcto y aplicarlo de forma inteligente — un enforcement demasiado estricto puede bloquear usuarios legítimos, mientras que un enforcement insuficiente invita al abuso; la estrategia ganadora es enforcement basado en riesgo, no bloqueo generalizado. Bloquear duro con `MEETS_STRONG_INTEGRITY` obligatorio en cada request expulsa usuarios legítimos con bootloader desbloqueado por razones normales (devs, ROMs custom, dispositivos viejos).

**Nivel de veredicto según severidad de la acción:** no todo necesita el mismo rigor — usar `deviceIntegrity` básico para leer datos, reservar `MEETS_STRONG_INTEGRITY` (attestation con hardware) para acciones de alto valor como pagos o recuperación de cuenta.

**Trade-off real:** agrega latencia (hay que ir y volver a servidores de Google) y una dependencia de red extra en el flujo crítico. No es gratis en UX — hay que diseñar qué pasa si el veredicto tarda o falla (fail-open vs fail-closed es una decisión de producto, no solo técnica).

## 5. Caso trampa

"Si el veredicto dice que el dispositivo no es íntegro, bloqueo la acción en el cliente" — esto es un error de diseño: **el cliente nunca debe ser quien decide** qué hacer con un veredicto de integridad, porque un cliente ya comprometido (justamente el escenario que estás tratando de detectar) puede simplemente saltearse esa lógica de bloqueo en el propio binario tampereado. La decisión de qué hacer con el veredicto (bloquear, degradar funcionalidad, pedir verificación extra) tiene que tomarla el **backend**, después de validar el token contra los servidores de Google — el cliente solo transporta el token, nunca interpreta el resultado por su cuenta.

Otro caso trampa: asumir que un dispositivo rooteado siempre es un usuario malicioso. Si tenés problemas con que un dispositivo de testing no cumpla la integridad de dispositivo, asegurate de que la ROM de fábrica esté instalada (por ejemplo, reseteando el dispositivo) y que el bootloader esté bloqueado. Hay devs y power users legítimos con bootloader desbloqueado — tratarlos igual que a un fraudster de leaderboard es sobre-enforcement que perjudica producto por una ganancia de seguridad marginal.

## 6. Conexión con arquitectura real (Timbax)

Timbax no implementa Play Integrity hoy — no hay pagos ni economía competitiva server-validada que lo justifique (coincide con el criterio de "NO usar" de la sección 4). Si en el futuro Timbax sumara un leaderboard global competitivo, el punto de entrada natural sería `Firebase App Check`, dado que el proyecto ya usa Firebase vía GitLive SDK: App Check se integraría como una capa extra sobre el `HttpClient`/llamadas a Firebase existentes, sin tener que armar la integración de Play Integrity desde cero. El patrón de implementación seguiría la misma lógica de `expect/actual` que ya se usa en `05_platform/expect_actual.md` — la obtención del token es una capacidad de plataforma (Android-only acá), mientras que el `UseCase` que orquesta la verificación vive en `domain`, agnóstico de si el chequeo vino de Play Integrity o de cualquier otro proveedor de attestation.