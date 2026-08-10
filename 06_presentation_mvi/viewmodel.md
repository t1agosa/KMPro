# viewmodel.md

## 1. Qué es

El **ViewModel** es la clase que orquesta una pantalla dentro del patrón MVI: recibe `Event` desde la UI a través de `onEvent()`, decide qué `UseCase` invocar según ese evento, actualiza el `State` con el resultado, y eventualmente emite un `Effect`. Vive en la capa Presentation y es el único punto de la app que conoce tanto el Contract de la pantalla (`06_presentation_mvi`) como los UseCases de Domain (`02_domain`) — nunca al revés.

No contiene lógica de negocio propia. Su trabajo es **orquestar y traducir**: traduce un Event de usuario en una llamada a UseCase, y traduce el resultado del UseCase (típicamente un `Result<T>`) en una actualización de State.

## 2. El problema que resuelve

Sin un ViewModel bien delimitado, la lógica de una pantalla termina repartida en dos lugares equivocados:

- **En el Composable**: si el `@Composable` llama directamente a un Repository o UseCase, se rompe la separación de capas (UI depende de Domain sin pasar por Presentation) y se pierde la capacidad de testear esa lógica sin levantar Compose.
- **En el UseCase**: si el UseCase empieza a manejar `isLoading` o a decidir cuándo mostrar un snackbar, se contamina con lógica de presentación que no le pertenece — un UseCase debería poder reusarse desde cualquier pantalla sin arrastrar detalles de cómo se muestra en una UI específica.

El ViewModel es la pieza que absorbe toda la lógica de "cómo reflejar esto en pantalla" (loading, mapeo de errores a mensajes, combinar resultados de varios UseCases) sin que esa responsabilidad se filtre hacia arriba (UI) ni hacia abajo (Domain).

## 3. Ejemplo mínimo comentado

```kotlin
// PlayersViewModel.kt

class PlayersViewModel(
    private val getPlayersUseCase: GetPlayersUseCase,
    private val deletePlayerUseCase: DeletePlayerUseCase
) {
    private val _state = MutableStateFlow(PlayersState())
    val state: StateFlow<PlayersState> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<PlayersEffect>()
    val effect: SharedFlow<PlayersEffect> = _effect.asSharedFlow()

    private val viewModelScope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)

    init {
        onEvent(PlayersEvent.OnRefresh)
    }

    fun onEvent(event: PlayersEvent) {
        when (event) {
            PlayersEvent.OnRefresh -> loadPlayers()
            is PlayersEvent.OnPlayerClicked -> {
                viewModelScope.launch {
                    _effect.emit(PlayersEffect.NavigateToDetail(event.id))
                }
            }
            is PlayersEvent.OnDeleteClicked -> deletePlayer(event.id)
        }
    }

    private fun loadPlayers() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }

            when (val result = getPlayersUseCase()) {
                is Result.Success -> _state.update {
                    it.copy(players = result.data, isLoading = false)
                }
                is Result.Error -> _state.update {
                    it.copy(isLoading = false, error = result.exception.message)
                }
            }
        }
    }

    private fun deletePlayer(id: String) {
        viewModelScope.launch {
            when (deletePlayerUseCase(id)) {
                is Result.Success -> _effect.emit(PlayersEffect.ShowSnackbar("Jugador eliminado"))
                is Result.Error -> _effect.emit(PlayersEffect.ShowSnackbar("No se pudo eliminar"))
            }
        }
    }
}
```

## 4. Matriz de criterio

| Elemento | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **`viewModelScope` con `SupervisorJob`** | Siempre — es el scope estándar de cualquier ViewModel MVI | — (no hay caso válido para omitirlo) | Un fallo en un `launch` no cancela a los demás; a cambio, cada `launch` necesita su propio manejo de error (try/catch o mapeo del `Result`), porque el scope ya no lo hace por vos |
| **UseCase inyectado por constructor** | El ViewModel necesita ejecutar una regla de negocio (siempre) | El ViewModel accede a datos triviales sin regla de negocio real detrás (raro, pero si existe, se cuestiona si hace falta UseCase en absoluto — ver `02_domain/usecases.md`) | Testeable con `FakeUseCase`/`FakeRepository` sin mockear frameworks; a cambio, cada acción nueva de UI suele implicar un UseCase nuevo, más archivos |
| **`init { }` disparando la carga inicial** | La pantalla necesita datos apenas se crea el ViewModel (caso más común) | La carga depende de un parámetro que llega después de la construcción (ej: un `playerId` de navegación) — ahí se dispara desde el primer `Event` o desde el `Composable` vía `LaunchedEffect`, no desde `init` | Simplicidad: no hace falta que la UI dispare un evento inicial; a cambio, dificulta testear el ViewModel de forma aislada si `init` ya dispara una coroutine — hay que controlar el `TestDispatcher` desde el primer momento |

## 5. Caso trampa

**"El ViewModel llama a dos UseCases en `loadPlayers()`, ¿está bien que el segundo dependa del resultado del primero?"**

La respuesta obvia — "sí, es normal encadenar llamadas" — esconde una trampa de diseño frecuente. Si `SaveScoreUseCase` necesita el resultado de `GetActivePlayerUseCase` para saber a quién guardarle el puntaje, encadenar las dos llamadas dentro del ViewModel (`getActivePlayerUseCase()` y después `saveScoreUseCase(player.id, score)`) es habitual y está bien — es orquestación legítima, la responsabilidad del ViewModel.

La trampa aparece cuando esa cadena empieza a *decidir* algo (por ejemplo: "si el jugador activo tiene más de 100 puntos, aplicar bonus antes de guardar"). Esa decisión es una **regla de negocio**, no orquestación — y si vive suelta en el ViewModel en vez de en un UseCase (`SaveScoreWithBonusUseCase`, o una validación dentro de `SaveScoreUseCase`), se rompe la garantía de que toda regla de negocio está en Domain, testeable sin mockear ViewModel ni Compose. La pregunta que desambigua: *¿esto es "en qué orden llamo las cosas" (orquestación, va en ViewModel) o "qué determina el resultado correcto" (regla, va en UseCase)?*

## 6. Conexión con arquitectura real

En Timbax, `PlayersViewModel` es exactamente el patrón del ejemplo: recibe `GetPlayersUseCase` y otros UseCases inyectados vía Koin con scope `viewModel` (ver `04_di/koin_fundamentos_y_scopes.md`), nunca instancia un `PlayerRepositoryImpl` directamente. El `Result` que devuelve cada UseCase es el mismo `sealed interface Result<out T>` de `02_domain/result_pattern.md` — el ViewModel es, de hecho, el único lugar de toda la app donde ese `Result` se "abre" con un `when` para decidir cómo actualizar el `State`; a partir de ahí, en toda capa hacia arriba (Compose), solo existe `PlayersState`, nunca un `Result` suelto.