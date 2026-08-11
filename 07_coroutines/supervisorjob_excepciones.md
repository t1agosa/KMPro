# 07_coroutines / `supervisorjob_excepciones.md`

---

## 1. Qué es

Por default, un `Job` normal propaga las excepciones de forma agresiva: si una coroutine hija lanza una excepción no capturada, **cancela a todas sus hermanas** y se propaga hacia el `Job` padre. `SupervisorJob` es una variante donde el fallo de una hija **no** cancela a sus hermanas ni se propaga hacia arriba — cada hija falla de forma aislada, como si fueran tareas independientes bajo el mismo scope.

`viewModelScope` usa internamente `SupervisorJob() + Dispatchers.Main.immediate`, justo por este motivo: un error cargando `stats` no debería tirar abajo la carga de `players` que corre en paralelo en la misma pantalla.

## 2. El problema que resuelve

Con un `Job` normal, este código es una bomba de tiempo:

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main) // Job normal, no Supervisor

scope.launch { repository.getPlayers() }  // si esto falla...
scope.launch { repository.getStats() }    // ...esto se cancela también, aunque no tenga nada que ver
```

Si `getPlayers()` lanza una excepción, el `Job` padre la propaga y cancela **todo** el scope — incluida la coroutine de `getStats()`, que ni siquiera falló. En una pantalla con varias secciones cargando datos independientes (dashboard con múltiples widgets, por ejemplo), esto significa que un solo error tumba toda la pantalla en vez de solo la sección afectada.

`SupervisorJob` resuelve esto aislando el fallo: cada `launch` bajo un `SupervisorJob` es responsable de su propio destino, y un error en uno no contamina a los demás.

## 3. Ejemplo mínimo comentado

```kotlin
class DashboardViewModel(
    private val getPlayers: GetPlayersUseCase,
    private val getStats: GetStatsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(DashboardState())
    val state: StateFlow<DashboardState> = _state.asStateFlow()

    fun onEvent(event: DashboardEvent.OnLoad) {
        // viewModelScope ya trae SupervisorJob internamente, no hace falta declararlo
        viewModelScope.launch {
            try {
                val players = getPlayers()
                _state.update { it.copy(players = players) }
            } catch (e: Exception) {
                _state.update { it.copy(playersError = e.message) }
            }
        }

        viewModelScope.launch {
            try {
                val stats = getStats()
                _state.update { it.copy(stats = stats) }
            } catch (e: Exception) {
                _state.update { it.copy(statsError = e.message) }
                // este catch atrapa el error solo porque el scope está protegido con SupervisorJob;
                // si getPlayers() falla en el otro launch, esta coroutine sigue viva
            }
        }
    }
}
```

Cada `launch` tiene su propio `try/catch` — eso maneja el error *dentro* de esa coroutine puntual. Lo que `SupervisorJob` garantiza por separado es que, aunque uno de los dos `launch` fallara sin `try/catch` (excepción no capturada), el otro `launch` no se cancelaría por eso.

## 4. Matriz de criterio

| Escenario | Qué usar | Por qué |
|---|---|---|
| Varias operaciones independientes en paralelo en la misma pantalla (dashboard con secciones separadas) | `SupervisorJob` (ya viene incluido en `viewModelScope`) | El fallo de una sección no debería tumbar las demás |
| Una secuencia de pasos donde el segundo depende del éxito del primero | `Job` normal (comportamiento por default fuera de `viewModelScope`) | Si el primer paso falla, tiene sentido que se cancele todo lo que dependía de él |
| Capturar el error de una `suspend fun` específica dentro de un `launch` | `try/catch` alrededor de la llamada | Forma más directa y predecible, funciona igual con o sin `SupervisorJob` |
| Capturar excepciones no manejadas de coroutines lanzadas con `launch`, a nivel scope | `CoroutineExceptionHandler` instalado en el scope | Útil como red de seguridad general, pero no reemplaza el `try/catch` puntual para manejar el error de forma específica (ej: mostrar un mensaje distinto según qué operación falló) |
| Un `async` que puede fallar | `try/catch` alrededor del `.await()` | La excepción queda "guardada" dentro del `Deferred` hasta que se llama `.await()`, ahí se relanza — `CoroutineExceptionHandler` no aplica igual a `async` |

**NO uses:**
- `CoroutineExceptionHandler` como único mecanismo de manejo de errores en un `ViewModel` — sirve como red de seguridad para excepciones realmente no anticipadas, pero no da forma de reflejar el error en el `State` de una sección específica de la UI; para eso siempre hace falta `try/catch` puntual.
- Un `Job()` normal a mano en un `ViewModel` en vez de confiar en `viewModelScope` — reintroducís el problema que `SupervisorJob` ya resuelve, sin necesidad.

## 5. Caso trampa

Un compañero reporta: *"mi app se cierra cuando falla una llamada de red, aunque tengo `try/catch` alrededor del `.await()`"*:

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main) // Job normal, no SupervisorJob

scope.launch {
    try {
        val players = async { repository.getPlayers() }
        val stats = async { repository.getStatsThatFails() } // esto lanza excepción
        _state.update { it.copy(players = players.await(), stats = stats.await()) }
    } catch (e: Exception) {
        _state.update { it.copy(error = e.message) } // nunca se ejecuta
    }
}
```

Se ve razonable: hay un `try/catch` envolviendo los dos `.await()`. Pero el `catch` **nunca se ejecuta**, y la app crashea igual. El motivo: con un `Job()` normal (no `SupervisorJob`), en el momento en que `getStatsThatFails()` lanza la excepción, esa excepción se propaga inmediatamente hacia arriba y **cancela todo el scope** — incluyendo la coroutine que contiene el `try/catch` — antes de que el flujo de ejecución llegue siquiera al bloque `catch`. El `try/catch` en el `.await()` solo funciona como uno espera si el scope está protegido con `SupervisorJob`; si no lo está, la cancelación del scope le gana de mano al `catch`.

La pregunta trampa de entrevista es exactamente esta: *"¿por qué mi `try/catch` no atrapa la excepción si está bien puesto?"* — la respuesta no está en el `try/catch` (está bien escrito), está en qué tipo de `Job` tiene el scope por debajo. `viewModelScope` no tiene este problema porque ya usa `SupervisorJob` internamente — el caso trampa aparece típicamente cuando alguien crea un `CoroutineScope` manual (fuera de `ViewModel`, en un test, en una clase propia) sin especificarlo.

## 6. Conexión con arquitectura real

En Timbax, el `ViewModel` de la pantalla de estadísticas de jugador combina la carga de `PlayerStats` (SQLDelight local) con la sincronización de `PlayerAchievements` (Firebase remoto) en dos `launch` separados dentro de `viewModelScope`. Ambas fuentes pueden fallar de forma completamente independiente — sin conexión a internet, `Achievements` falla pero `Stats` (local) carga bien — y gracias a que `viewModelScope` ya trae `SupervisorJob`, el `State` puede reflejar `stats` cargados con `achievementsError` seteado, en vez de que un problema de red tumbe toda la pantalla. Cada `launch` mantiene su propio `try/catch` para mapear la excepción puntual a su campo correspondiente del `State`, siguiendo el mismo patrón del punto 3.