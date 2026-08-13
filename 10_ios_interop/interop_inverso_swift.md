# Interop Inverso (Swift/Objective-C llamando a Kotlin)

## 1. Qué es

La dirección contraria a lo visto en `framework_xcframework_cinterop.md`: en vez de Kotlin llamando APIs nativas de iOS, acá es **Swift consumiendo directamente las clases y funciones públicas de tu módulo `shared`**. Todo lo público en `commonMain`/`iosMain` se expone automáticamente en el `.framework` generado como clases y funciones Objective-C/Swift-friendly — no hace falta escribir ningún wrapper manual para que existan del lado Swift, el compilador de Kotlin/Native se encarga de esa traducción.

El detalle importante está en **cómo se traducen dos construcciones específicas de Kotlin que no tienen equivalente directo en Swift**: `suspend fun` y `Flow`.

## 2. El problema que resuelve

Sin interop inverso, compartir lógica en KMP no serviría de mucho — podrías tener toda tu `domain`/`data`/`presentation` en Kotlin puro, pero si Swift no pudiera consumirla de forma razonable, el equipo iOS terminaría reescribiendo esa lógica en Swift de todos modos, anulando el propósito de compartir código.

El problema específico que resuelve (parcialmente, con ayuda) es la brecha de paradigmas asincrónicos: Kotlin tiene coroutines (`suspend fun`, `Flow`) y Swift tiene su propio modelo (`async/await`, Combine/`AsyncSequence`). No son binariamente compatibles — Objective-C, que es el puente real que usa Kotlin/Native para exponerse, no tiene concepto de coroutine. Por eso el compilador traduce `suspend fun` a algo que Objective-C sí entiende (completion handlers/callbacks), y `Flow` queda directamente sin traducción automática — necesita una librería puente.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — código compartido típico de Timbax
class PlayerRepository {
    suspend fun getPlayers(): List<Player> { /* ... */ }
    fun observePlayers(): Flow<List<Player>> { /* ... */ }
}
```

```swift
// Swift, SIN librería puente — así se ve el suspend fun expuesto
// Kotlin/Native lo traduce a una función con completion handler (callback)
sharedRepository.getPlayers { players, error in
    if let players = players {
        // actualizar UI con los datos
    } else if let error = error {
        // manejar el error
    }
}

// el Flow NO tiene equivalente directo — esto NO compila así en Swift plano.
// sin librería puente, tendrías que suscribirte manualmente a un objeto
// Closeable/Cancellable que Kotlin/Native genera para representar la suscripción.
```

```swift
// Swift, CON SKIE o KMP-NativeCoroutines instalado —
// el suspend fun se consume con async/await nativo de Swift
let players = try await sharedRepository.getPlayers()

// y el Flow se expone como AsyncSequence, iterable con for-await
for await players in sharedRepository.observePlayers() {
    // actualizar UI en cada nueva emisión, de forma idiomática
}
```

## 4. Matriz de criterio

| Usar cuando | NO aplica cuando | Trade-off real |
|---|---|---|
| El equipo iOS necesita consumir `suspend fun` de forma idiomática (`async/await`) en vez de completion handlers anidados | El proyecto es KMP con muy poca superficie async expuesta a iOS (por ejemplo, solo modelos de datos y funciones síncronas) — ahí el interop por defecto ya alcanza sin agregar dependencias | Agregar SKIE o KMP-NativeCoroutines suma una dependencia más al build (plugin de Gradle + configuración), y ambas requieren mantenerse actualizadas junto con la versión de Kotlin |
| El `ViewModel`/`Repository` expone `StateFlow`/`Flow` que la UI de SwiftUI necesita observar reactivamente | El equipo iOS prefiere manejar la suscripción manual al `Closeable` que genera Kotlin/Native por control explícito del ciclo de vida (menos común, pero válido en proyectos muy chicos) | SKIE es más automático (genera la API Swift-friendly directo desde el plugin), KMP-NativeCoroutines requiere un paso de generación de código adicional — cada uno tiene su propia curva de configuración |

## 5. Caso trampa

**Pregunta:** *"Ya expuse mi `PlayersViewModel` con su `StateFlow<PlayersState>` al framework de iOS. El equipo iOS dice que no puede usar `for await` sobre el `state`. ¿Está mal expuesto del lado Kotlin?"*

Respuesta ingenua: "Raro, debería funcionar solo, es un `StateFlow` público."

Respuesta correcta: no está mal expuesto — es el comportamiento esperado **sin** una librería puente. `Flow`/`StateFlow` no tienen traducción automática a `AsyncSequence` de Swift por parte del compilador de Kotlin/Native; a diferencia de `suspend fun` (que sí se traduce solo, aunque de forma poco idiomática, a completion handlers), `Flow` directamente no se expone como algo consumible con sintaxis nativa de Swift sin agregar SKIE o KMP-NativeCoroutines al proyecto. La trampa es asumir que "si `suspend fun` se resuelve solo, `Flow` también" — son casos distintos: uno tiene traducción built-in (aunque fea), el otro no tiene traducción built-in en absoluto.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el `PlayersViewModel` expone `state: StateFlow<PlayersState>` como única fuente de verdad de la pantalla (patrón MVI ya documentado en `06_presentation_mvi`). Si el proyecto sumara una versión iOS, ese mismo `StateFlow` sería el punto de fricción esperable del interop inverso: sin SKIE/KMP-NativeCoroutines, el equipo iOS tendría que suscribirse manualmente con un callback y gestionar la cancelación a mano; con la librería puente, `state` se consumiría de forma casi idéntica a un `@Published` de Combine, integrándose naturalmente con SwiftUI sin que el `ViewModel` compartido necesite ningún cambio.