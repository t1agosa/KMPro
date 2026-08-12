# navegacion.md

## 1. Qué es

La navegación en Compose Multiplatform es el mecanismo por el cual la app transiciona entre distintas pantallas (`@Composable` de nivel "screen"), manteniendo un back stack (pila de pantallas visitadas) que permite avanzar y retroceder. `Navigation Compose` (la librería de Jetpack, con soporte multiplatform oficial en versiones recientes) es la opción más común hoy; `Voyager` y `Decompose` son alternativas que nacieron pensadas para KMP desde el principio.

Pero la parte más importante de este archivo no es "qué librería usar" — es **cómo se conecta la navegación con MVI**: un botón nunca debería llamar a `navController.navigate(...)` directamente. La navegación se dispara como reacción a un `Effect` que viene del `ViewModel`.

## 2. El problema que resuelve

Si un composable llamara a `navController.navigate("detail/$id")` directamente dentro de un `onClick`, la decisión de "a dónde navegar" quedaría mezclada con la capa de UI — imposible de testear sin renderizar composables, y rompiendo la separación de MVI donde toda lógica (incluida "qué pasa después de esta acción") debería poder razonarse desde el `ViewModel`, sin depender de un framework de UI específico.

El patrón correcto resuelve esto tratando la navegación como un **Effect** más (documentado en `08_flow/sharedflow_channel.md`): algo que ocurre una sola vez, no es parte del estado persistente, y se dispara desde el `ViewModel` en respuesta a una regla de negocio o de flujo — el composable solo escucha ese `Effect` y llama al `navController` como reacción, sin decidir nada por su cuenta.

## 3. Ejemplo mínimo comentado

```kotlin
// Contract: la navegación es un caso más del Effect sellado
sealed interface PlayersEffect {
    data class NavigateToPlayerDetail(val playerId: String) : PlayersEffect
    data class ShowSnackbar(val message: String) : PlayersEffect
}

// ViewModel: decide CUÁNDO navegar, sin saber CÓMO se navega
class PlayersViewModel(
    private val getPlayersUseCase: GetPlayersUseCase
) : ViewModel() {
    private val _effect = MutableSharedFlow<PlayersEffect>()
    val effect: SharedFlow<PlayersEffect> = _effect.asSharedFlow()

    fun onEvent(event: PlayersEvent) {
        when (event) {
            is PlayersEvent.OnPlayerClicked -> {
                viewModelScope.launch {
                    _effect.emit(PlayersEffect.NavigateToPlayerDetail(event.playerId))
                }
            }
            // ...
        }
    }
}

// Screen: colecta el Effect y ejecuta la navegación real (el único lugar que conoce navController)
@Composable
fun PlayersScreen(
    viewModel: PlayersViewModel,
    navController: NavController
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is PlayersEffect.NavigateToPlayerDetail ->
                    navController.navigate("player_detail/${effect.playerId}")
                is PlayersEffect.ShowSnackbar -> { /* ... */ }
            }
        }
    }

    PlayersList(
        players = state.players,
        onPlayerClick = { id -> viewModel.onEvent(PlayersEvent.OnPlayerClicked(id)) }
    )
}
```

El loop completo: click del usuario → `Event` hacia el `ViewModel` → el `ViewModel` decide que corresponde navegar y emite un `Effect` → `LaunchedEffect(Unit)` (documentado en `effects_guia_completa.md`) lo colecta → recién ahí se llama a `navController.navigate()`. `PlayerCard`/`PlayersList` nunca supieron que existía un `NavController`.

## 4. Matriz de criterio

**Navegación vía `Effect` vs navegación directa en el composable**
- Usar `Effect` cuando: siempre, en cualquier navegación que dependa de una acción del usuario o una regla de negocio (login exitoso → ir a home, guardar partida → volver a la lista) — es la única forma de mantener esa decisión testeable en el `ViewModel` sin depender de Compose.
- NO llamar `navController.navigate()` directamente desde un `onClick` — funciona, pero mezcla capas: la decisión de "a dónde ir" queda atrapada en la UI, sin poder testearse con un test de `ViewModel` puro (`fakes_vs_mocks_turbine.md`).
- Trade-off: el patrón vía `Effect` agrega una vuelta extra de indirección (Event → Effect → LaunchedEffect → navigate) comparado con llamar `navigate()` directo — a cambio de testeabilidad real y consistencia con el resto de MVI.

**`Navigation Compose` vs `Voyager`/`Decompose`**
- Usar `Navigation Compose` cuando: el proyecto es Android-first con KMP como plus — reduce fricción por familiaridad con el ecosistema Google, y hoy tiene soporte multiplatform oficial en versiones recientes.
- Usar `Voyager`/`Decompose` cuando: el proyecto es KMP-first (verdaderamente multiplataforma desde el diseño) — suelen ofrecer mejor manejo de ciclo de vida multiplataforma o navegación type-safe más madura en iOS/Desktop.
- Trade-off: esta decisión no afecta el patrón de este archivo — sea cual sea la librería, la navegación se sigue disparando como reacción a un `Effect`, nunca directo desde el composable.

**Argumentos de navegación (pasar `playerId` entre pantallas)**
- Usar cuando: se pasan solo identificadores livianos (`playerId: String`) como argumento de ruta — la pantalla destino vuelve a pedir los datos completos a su propio `ViewModel`/`UseCase` usando ese id.
- NO pasar cuando: se intenta pasar el objeto `Player` completo como argumento de navegación — generalmente implica serialización innecesaria y acopla la navegación a la forma exacta del modelo de dominio; mejor volver a resolver el dato completo en destino a partir del id.

## 5. Caso trampa

```kotlin
@Composable
fun PlayerCard(player: Player, navController: NavController) {
    Card(
        modifier = Modifier.clickable {
            navController.navigate("player_detail/${player.id}") // navegación directa
        }
    ) {
        Text(player.name)
    }
}
```

La trampa: esto compila, funciona, y de hecho es el patrón que aparece en la mayoría de los tutoriales básicos de Compose — pasar el `navController` como parámetro hasta el composable hoja y llamarlo directo en el `onClick`. El problema aparece más tarde: si mañana la navegación a detalle necesita una condición de negocio (por ejemplo, "si el jugador tiene una partida en curso, mostrar antes un diálogo de confirmación"), esa regla tiene que agregarse *dentro del composable*, porque ahí es donde vive la decisión de navegar — mezclando lógica de negocio con código de UI, y haciendo que esa regla sea imposible de testear con un test de `ViewModel`. Además, `PlayerCard` deja de ser un composable "tonto" y reusable (`composables_y_state_hoisting.md`): ahora depende de un `NavController` concreto, no puede usarse en un contexto sin navegación (un preview, un test, otra pantalla que solo quiere mostrar la card sin navegar). La corrección es la del ejemplo de la sección 3: `PlayerCard` solo emite `onClick: (String) -> Unit`, y quien decide navegar es el `ViewModel`, vía `Effect`.

## 6. Conexión con arquitectura real

En Timbax, este archivo cierra el loop que empezó a documentarse en `06_presentation_mvi/contract_state_event_effect.md` ("Effect: algo que ocurre una sola vez... navegar, mostrar snackbar, vibrar") y se profundizó en `08_flow/sharedflow_channel.md` (por qué `SharedFlow(replay=0)` es la elección correcta para Effects) y en `effects_guia_completa.md` (`LaunchedEffect(Unit) { viewModel.effect.collect { ... } }` como punto de entrada). La navegación no es un caso especial fuera de MVI — es exactamente un `Effect` más, con la única particularidad de que quien lo consume (`PlayersScreen`) necesita tener acceso al `NavController`, algo que ningún composable hijo (`PlayerCard`) debería necesitar jamás.