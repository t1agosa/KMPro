# contract_state_event_effect.md

## 1. Qué es

El **Contract** es el archivo que define, para una sola pantalla, las tres cosas que en MVI están separadas por diseño:

- **State**: la foto completa de lo que la UI debe mostrar en este instante.
- **Event** (a veces llamado Intent): lo que el usuario puede hacer, entra al ViewModel.
- **Effect**: algo que ocurre una sola vez y no es parte del estado persistente de la pantalla.

No es una clase ni una interfaz que se instancie — es una convención de organización: los tres tipos (normalmente `data class` para State, `sealed interface` para Event y Effect) viven juntos en un mismo archivo, típicamente `NombrePantallaContract.kt`, como declaración explícita de "esto es todo lo que esta pantalla puede ser y hacer".

## 2. El problema que resuelve

Sin un Contract explícito, un ViewModel tiende a crecer de dos formas problemáticas:

- **Métodos públicos sueltos**: `viewModel.onPlayerClicked(id)`, `viewModel.onRefresh()`, `viewModel.onDeleteClicked(id)`. Cada uno es una entrada distinta, sin un lugar único donde ver "todo lo que el usuario puede hacer acá". Agregar una acción nueva significa agregar un método público más, sin que nada te obligue a documentarlo en un solo lugar.
- **Streams de estado sueltos**: `val isLoading: StateFlow<Boolean>`, `val players: StateFlow<List<Player>>`, `val error: StateFlow<String?>`. Esto habilita combinaciones imposibles (loading = true y error != null al mismo tiempo, por ejemplo) porque no hay nada que ate esos tres valores entre sí.

El Contract resuelve ambos problemas con una regla simple: **una sola entrada** (el `sealed interface Event`, procesado en un único `onEvent()`) y **una sola salida de estado persistente** (el `data class State`, expuesto como `StateFlow`). Todo lo que la pantalla necesita — y nada más — vive documentado en un solo archivo, legible de arriba a abajo antes de tocar una sola línea de implementación.

## 3. Ejemplo mínimo comentado

```kotlin
// PlayersContract.kt — todo lo que la pantalla de jugadores de Timbax puede ser y hacer

data class PlayersState(
    val players: List<Player> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

sealed interface PlayersEvent {
    data object OnRefresh : PlayersEvent
    data class OnPlayerClicked(val id: String) : PlayersEvent
    data class OnDeleteClicked(val id: String) : PlayersEvent
}

sealed interface PlayersEffect {
    data class ShowSnackbar(val message: String) : PlayersEffect
    data class NavigateToDetail(val playerId: String) : PlayersEffect
}
```

```kotlin
// Uso típico dentro del ViewModel (se profundiza en viewmodel.md)

class PlayersViewModel(
    private val getPlayersUseCase: GetPlayersUseCase
) {
    private val _state = MutableStateFlow(PlayersState())
    val state: StateFlow<PlayersState> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<PlayersEffect>()
    val effect: SharedFlow<PlayersEffect> = _effect.asSharedFlow()

    fun onEvent(event: PlayersEvent) {
        when (event) {
            PlayersEvent.OnRefresh -> loadPlayers()
            is PlayersEvent.OnPlayerClicked -> navigateToDetail(event.id)
            is PlayersEvent.OnDeleteClicked -> deletePlayer(event.id)
        }
    }
}
```

## 4. Matriz de criterio

| Elemento | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **State como `data class` único** | La pantalla tiene 2+ piezas de estado que interactúan entre sí (loading + data + error) | Pantalla trivial de un solo valor sin combinaciones posibles (ahí un `StateFlow<Boolean>` suelto puede alcanzar) | Ganás consistencia garantizada; perdés algo de granularidad — cualquier cambio de un campo dispara una nueva emisión completa del State |
| **Event como `sealed interface`** | Necesitás una superficie de entrada única, testeable con `(State, Event) -> State` | Prototipo descartable donde no vale la pena el boilerplate | Ganás exhaustividad en el `when` (el compilador te obliga a cubrir cada caso); perdés algo de velocidad inicial de escritura |
| **Effect como `SharedFlow(replay=0)`** | El evento debe consumirse una sola vez (snackbar, navegación, vibración) | El "evento" en realidad describe un estado persistente que la UI debería poder re-consultar (ahí va en State, no en Effect) | Evita que rotar pantalla repita un snackbar ya mostrado; a cambio, si nadie está coleccionando el Flow en el momento exacto de la emisión, ese Effect se pierde (comportamiento esperado, no bug) |

## 5. Caso trampa

**Un mensaje de error, ¿es State o Effect?**

La respuesta obvia — "es un error, va en Effect como el snackbar" — es incorrecta en la mayoría de los casos.

Si el error debe **persistir visible** hasta que el usuario lo descarte o reintente (por ejemplo, un texto de error debajo de un formulario, o un estado "no se pudieron cargar los jugadores" que sigue en pantalla si rotás el dispositivo), es **State** — porque necesita sobrevivir a una nueva suscripción (rotación, recomposición).

Si el error es un **aviso puntual** que no necesita quedar visible después de mostrarse (un snackbar de "no se pudo guardar, reintentá"), es **Effect**.

La pregunta que desambigua: *si el usuario rota la pantalla ahora mismo, ¿el error debería seguir viéndose?* Si sí → State. Si no → Effect. Confundir esto es el error más común al diseñar un Contract nuevo.

## 6. Conexión con arquitectura real

En Timbax, el Contract de cada pantalla es el punto de entrada que lee cualquiera que abra el archivo por primera vez — antes de mirar el ViewModel o la UI. `PlayersState` es lo único que consume el `@Composable` vía `collectAsStateWithLifecycle()` (ver `09_ui_compose`); `PlayersEvent` es lo único que el Composable puede enviar hacia arriba. El Effect conecta con `SharedFlow` (ver `08_flow/sharedflow_channel.md`) — la razón técnica de por qué se usa `SharedFlow(replay=0)` y no `StateFlow` para esta parte del Contract está desarrollada ahí en detalle.