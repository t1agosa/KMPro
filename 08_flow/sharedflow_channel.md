# SharedFlow y Channel

## 1. Qué es

`SharedFlow<T>` es similar a `StateFlow` pero configurable en cuánto "recuerda" (parámetro `replay`). Con `replay = 0`, un suscriptor nuevo no recibe nada de lo emitido antes de suscribirse, solo emisiones futuras. `Channel` es una cola de comunicación entre coroutines donde cada valor emitido se entrega a un único consumidor (point-to-point), a diferencia de Flow/SharedFlow que son multicast (varios colectores pueden recibir la misma emisión).

## 2. El problema que resuelve

`StateFlow` no sirve para modelar eventos que deben consumirse **una sola vez** (navegar, mostrar snackbar, vibrar), porque siempre repite su último valor a cualquier nuevo suscriptor. `SharedFlow(replay=0)` (o `Channel`) resuelve exactamente ese hueco: emite solo hacia adelante, sin memoria de lo pasado, así que una rotación de pantalla que resuscribe todo no reproduce un efecto ya consumido.

## 3. Ejemplo mínimo comentado

```kotlin
class PlayersViewModel(
    private val saveScoreUseCase: SaveScoreUseCase
) : ViewModel() {

    private val _effect = MutableSharedFlow<PlayersEffect>() // replay=0 por default
    val effect: SharedFlow<PlayersEffect> = _effect.asSharedFlow()

    fun onEvent(event: PlayersEvent) {
        when (event) {
            is PlayersEvent.OnSaveClicked -> viewModelScope.launch {
                saveScoreUseCase(event.playerId, event.score)
                _effect.emit(PlayersEffect.ShowSnackbar("Guardado")) // se consume una sola vez
            }
        }
    }
}
```

```kotlin
// consumo en la UI (Compose), típicamente dentro de un LaunchedEffect(Unit)
LaunchedEffect(Unit) {
    viewModel.effect.collect { effect ->
        when (effect) {
            is PlayersEffect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
        }
    }
}
```

## 4. Matriz de criterio

| Usar cuando | NO usar cuando |
|---|---|
| `SharedFlow(replay=0)`: Effects multicast — puede haber más de un colector legítimo (UI real + un test con Turbine, por ejemplo) | Necesitás que el suscriptor reciba el último valor al conectarse (ahí es `StateFlow`) |
| `Channel`: comunicación point-to-point real, donde por diseño solo debe existir un consumidor | Tenés potencialmente más de un colector y necesitás que todos reciban el mismo evento (`Channel` reparte, no multicastea) |

**Trade-off real:** hoy `SharedFlow(replay=0)` es la opción más común para Effects en KMP, en parte porque integra mejor con `collectAsStateWithLifecycle` y el ecosistema de Compose. `Channel` sigue siendo válido pero menos elegido para este caso específico.

## 5. Caso trampa

Migrás un Effect de `Channel` a `SharedFlow` pensando que es un cambio "cosmético", sin cambiar nada más. Pero si en algún punto tenías dos colectores del mismo Channel esperando "repartirse" los eventos (por ejemplo, un test y la UI real corriendo en paralelo durante debug), con `SharedFlow` de golpe **ambos** reciben cada emisión — porque `SharedFlow` es multicast y `Channel` es point-to-point. Un comportamiento que "funcionaba por casualidad" con Channel (un evento, un consumidor) se rompe silenciosamente al cambiar el tipo, porque ahora el evento le llega a todos.

## 6. Conexión con arquitectura real

En Timbax, `PlayersEffect` (parte del contrato definido en `06_presentation_mvi/contract_state_event_effect.md`) se expone como `SharedFlow<PlayersEffect>` desde el ViewModel. La UI lo colecta dentro de un `LaunchedEffect(Unit)` (ver `09_ui_compose/effects_guia_completa.md`), nunca como parte del `State` — es la contraparte exacta de lo documentado en `stateflow.md` sobre por qué el Effect no puede ser `StateFlow`.