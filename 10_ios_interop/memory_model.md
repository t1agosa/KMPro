# Memory Model (Modelo de Memoria)

## 1. Qué es

La estrategia que usa Kotlin/Native para permitir que objetos mutables se compartan y muten entre distintos threads. Antes de Kotlin 1.7.20, existía un modelo "estricto" donde un objeto mutable creado en un thread no podía mutarse desde otro thread sin antes "congelarlo" (`freeze()`) — convertirlo en inmutable en runtime. Desde 1.7.20 (hoy el único modelo soportado en versiones recientes), ese requisito desapareció: los objetos se comparten y mutan entre threads igual que en la JVM, sin `freeze()` manual.

Es un tema que hoy es mayormente **histórico** — importa para entender por qué código KMP viejo tiene `freeze()` por todos lados, y para no repetir en una entrevista un miedo que ya no aplica.

## 2. El problema que resuelve (o que resolvía)

El modelo viejo generaba fricción real y errores confusos en runtime. Un caso típico: un `ViewModel` corriendo una coroutine en background (por ejemplo, en `Dispatchers.Default` o `Dispatchers.IO`) que intentaba actualizar un objeto mutable desde ese thread, después de que ese mismo objeto hubiera cruzado de threads en algún punto anterior — el runtime lo había congelado implícitamente, y el intento de mutarlo lanzaba `InvalidMutabilityException` en tiempo de ejecución, no en compilación. Era un bug que aparecía tarde, difícil de razonar, y específico de iOS (en Android el mismo código corría sin problema, porque Kotlin/JVM nunca tuvo esa restricción).

El nuevo modelo de memoria (1.7.20+) elimina el problema de raíz: ya no existe el concepto de "congelar" un objeto al cruzar de thread. El comportamiento de compartir/mutar estado entre threads pasa a ser consistente entre Android y iOS, sin ese cuidado adicional.

## 3. Ejemplo mínimo comentado

```kotlin
// ViewModel típico en Timbax, cargando datos en paralelo (patrón ya visto en 07_coroutines)
class PlayersViewModel(
    private val getPlayers: GetPlayersUseCase,
    private val getStats: GetStatsUseCase
) {
    private val _state = MutableStateFlow(PlayersState())
    val state: StateFlow<PlayersState> = _state.asStateFlow()

    fun loadDashboard() {
        viewModelScope.launch {
            // con el modelo de memoria ANTIGUO (pre 1.7.20):
            // si "players" y "stats" cruzaban de thread en el async,
            // el objeto podía llegar "congelado" y el .copy() de más abajo
            // podía lanzar InvalidMutabilityException en iOS (nunca en Android)

            // con el modelo de memoria ACTUAL (1.7.20+):
            // esto simplemente funciona igual en Android y en iOS
            val players = async { getPlayers() }
            val stats = async { getStats() }
            _state.update { it.copy(players = players.await(), stats = stats.await()) }
        }
    }
}
```

No hay ninguna línea de código especial que "activa" el nuevo modelo — es el comportamiento por defecto del compilador en versiones recientes de Kotlin. El ejemplo de arriba simplemente *no tiene* el problema que sí tendría en una versión vieja del compilador.

## 4. Matriz de criterio

| Usar cuando | NO aplica cuando | Trade-off real |
|---|---|---|
| Estás en un proyecto Kotlin/Native moderno (post 1.7.20) — que es prácticamente cualquier proyecto KMP activo hoy | Estás manteniendo un proyecto legacy con Kotlin muy viejo (pre 1.7.20) — ahí sí podés encontrarte con `freeze()` real en el código | Ninguno práctico hoy: el nuevo modelo es estrictamente una mejora, sin trade-off de performance ni de comportamiento a cambiar |
| Te preguntan en una entrevista sobre concurrencia en Kotlin/Native | Estás depurando un bug de concurrencia genuino (race condition) — el nuevo modelo de memoria no elimina la necesidad de sincronización correcta, solo eliminó el `freeze()` obligatorio | — |

## 5. Caso trampa

**Pregunta:** *"Estoy viendo un tutorial/repo viejo de KMP que tiene `.freeze()` en varios lados del código compartido. ¿Tengo que agregar eso también en Timbax para que funcione bien en iOS?"*

Respuesta ingenua: "Sí, mejor agrego `freeze()` por las dudas, total no rompe nada."

Respuesta correcta: **no**, con el modelo de memoria actual `.freeze()` es innecesario y, más importante, es una señal de que el tutorial/repo que estás mirando está desactualizado (pre-1.7.20). Agregarlo hoy no rompe el código, pero es ruido conceptual — estás resolviendo un problema que el compilador moderno ya no tiene, y puede confundirte (o confundir a quien lea tu código) haciendo pensar que sigue siendo necesario. La señal de alarma real acá es aprender a **reconocer que una fuente está desactualizada** por este tipo de detalle, algo que aplica más allá de este caso puntual: si un tutorial de KMP menciona `freeze()`, `AtomicReference` como workaround manual, o `InvalidMutabilityException`, probablemente esté describiendo el modelo viejo.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el `ViewModel` usa `viewModelScope` con coroutines que cruzan de `Dispatchers.Main` a `Dispatchers.IO` constantemente (por ejemplo, al llamar `SaveScoreUseCase` desde un evento de UI y actualizar el `State` con el resultado). Con el modelo de memoria actual, ese flujo funciona idéntico en Android y en una futura versión iOS, sin ningún cuidado extra en cómo se comparten los objetos entre threads — el mismo código de `PlayersViewModel` que ya corre en Android sería, en este aspecto puntual, directamente portable sin modificaciones.