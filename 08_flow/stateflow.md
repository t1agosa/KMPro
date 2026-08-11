# StateFlow

## 1. Qué es

`StateFlow<T>` es un `Flow` especializado, **caliente** (hot) y con estado: siempre tiene un valor actual, y cualquier suscriptor nuevo recibe inmediatamente el último valor emitido, sin importar cuándo se suscribió.

## 2. El problema que resuelve

La UI necesita poder pintar "el estado actual" apenas se suscribe — primera carga de pantalla, rotación, recomposición — sin tener que esperar a que llegue una nueva emisión. Un `Flow` común (frío) no resuelve esto: si nadie está colectando en el momento exacto de una emisión, esa emisión se pierde para un suscriptor que llega después. `StateFlow` garantiza que siempre hay "algo" para mostrar de inmediato.

## 3. Ejemplo mínimo comentado

```kotlin
class PlayersViewModel(
    private val getPlayersUseCase: GetPlayersUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(PlayersState()) // valor inicial obligatorio
    val state: StateFlow<PlayersState> = _state.asStateFlow() // expuesto como solo-lectura

    init {
        viewModelScope.launch {
            getPlayersUseCase().collect { players ->
                _state.update { it.copy(players = players, isLoading = false) }
            }
        }
    }
}
```

## 4. Matriz de criterio

| Usar cuando | NO usar cuando |
|---|---|
| El consumidor necesita "el estado actual" en cualquier momento en que se suscriba (el `State` de MVI) | Es un evento de una sola vez que no debería repetirse si alguien se vuelve a suscribir (ahí es `Effect`, ver `sharedflow_channel.md`) |
| Necesitás que Compose pueda comparar el valor actual contra el anterior para decidir si recomponer | El valor no tiene sentido como "estado persistente" (por ejemplo, un click puntual) |

**Trade-off real:** `StateFlow` siempre necesita un valor inicial (no hay forma de crear uno vacío), y por diseño **repite** el último valor a cada nuevo suscriptor — esto es justamente lo que lo hace inútil para Effects.

## 5. Caso trampa

Usás `StateFlow` para exponer un evento tipo "mostrar snackbar de guardado exitoso", pensando que es "solo otro stream más del ViewModel". El usuario guarda el score, ve el snackbar, y después rota la pantalla. Como la rotación fuerza una nueva recomposición y una nueva suscripción al `StateFlow`, el snackbar se vuelve a mostrar — aunque el usuario no volvió a guardar nada. El bug no está en la UI, está en haber elegido el tipo de stream equivocado para ese caso.

## 6. Conexión con arquitectura real

En Timbax, `PlayersState` (definido en `06_presentation_mvi/contract_state_event_effect.md`) es el `data class` inmutable expuesto como `StateFlow<PlayersState>` desde el ViewModel — es la única fuente de verdad que consume la UI vía `collectAsStateWithLifecycle()` (ver `09_ui_compose`). Todo lo que puede pasar en la pantalla vive ahí adentro: `players`, `isLoading`, etc.