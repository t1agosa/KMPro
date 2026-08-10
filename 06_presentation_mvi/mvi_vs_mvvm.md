# mvi_vs_mvvm.md

## 1. Qué es

**MVI** (Model-View-Intent) y **MVVM** (Model-View-ViewModel) son dos formas de organizar la capa Presentation. Ambos usan un ViewModel que separa lógica de presentación de la UI, y ambos exponen estado observable (`StateFlow`/`LiveData`) para que la UI reaccione sin que el ViewModel conozca detalles de Compose o de la plataforma. La diferencia real no está en "si hay ViewModel o no" — los dos lo tienen. Está en **cómo se modela el estado que ese ViewModel expone** y **cómo entran las acciones del usuario**.

- **MVVM "clásico"**: el ViewModel expone varios streams sueltos, uno por cada pieza de estado, y la UI suele llamar métodos públicos directos del ViewModel para cada acción.
- **MVI**: el ViewModel expone un único `State` inmutable que agrupa todo, y las acciones entran por un único canal (`Event`/`Intent`).

## 2. El problema que resuelve

En MVVM clásico, cada pieza de estado vive en su propio `StateFlow` independiente:

```kotlin
class PlayersViewModel {
    val isLoading: StateFlow<Boolean> = ...
    val players: StateFlow<List<Player>> = ...
    val error: StateFlow<String?> = ...
}
```

El problema de fondo: estos streams no tienen ninguna relación entre sí a nivel de tipos. Nada impide que, en un momento dado, `isLoading = true`, `players` ya tenga datos cargados de una carga anterior, **y** `error` tenga un mensaje — los tres al mismo tiempo. Esa combinación no debería poder existir nunca, pero el propio modelo de datos no la prohíbe. La UI (o el ViewModel) termina necesitando lógica extra tipo `if (isLoading && error != null && players.isNotEmpty())` para desambiguar casos que, en rigor, nunca deberían haber podido coexistir.

MVI resuelve esto agrupando todo en un único `data class State`: cualquier combinación de campos que exista ahí es, por construcción, la representación completa y consistente de "qué está pasando en la pantalla en este instante". No hay forma de que la UI reciba una actualización parcial de un stream sin las otras — siempre llega la foto completa (ver `contract_state_event_effect.md`, sección "State: única fuente de verdad").

## 3. Ejemplo mínimo comentado

```kotlin
// MVVM clásico — streams sueltos, sin relación entre sí a nivel de tipos
class PlayersViewModelMvvm {
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()

    private val _players = MutableStateFlow<List<Player>>(emptyList())
    val players: StateFlow<List<Player>> = _players.asStateFlow()

    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()

    // acciones: métodos públicos sueltos, sin canal único
    fun onPlayerClicked(id: String) { /* ... */ }
    fun onRefresh() { /* ... */ }
}
```

```kotlin
// MVI — un único State, un único canal de Event
data class PlayersState(
    val players: List<Player> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class PlayersViewModelMvi {
    private val _state = MutableStateFlow(PlayersState())
    val state: StateFlow<PlayersState> = _state.asStateFlow()

    fun onEvent(event: PlayersEvent) {
        when (event) {
            is PlayersEvent.OnPlayerClicked -> selectPlayer(event.id)
            PlayersEvent.OnRefresh -> loadPlayers()
        }
    }
}
```

```kotlin
// El corazón testeable de MVI: un reducer puro (State, Event) -> State
// Esto se testea sin mockear ViewModel ni UI — ver 12_testing
fun reduce(current: PlayersState, event: PlayersEvent): PlayersState = when (event) {
    is PlayersEvent.OnPlayerClicked -> current.copy(selectedId = event.id)
    PlayersEvent.OnRefresh -> current.copy(isLoading = true)
}
```

## 4. Matriz de criterio

| Elemento | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **MVVM clásico (streams sueltos)** | Pantalla muy simple, con poco estado y sin combinaciones posibles (ej: un formulario de 2 campos sin loading/error cruzados) | La pantalla tiene loading + data + error + paginación interactuando entre sí | Menos boilerplate inicial (no hace falta Contract, ni reducer); a cambio, no hay garantía de consistencia — el bug de "estados imposibles" queda a cargo de la disciplina del dev, no del compilador |
| **MVI (State único + Event)** | La pantalla tiene 2+ piezas de estado que interactúan, o el proyecto ya adoptó MVI como estándar de equipo (consistencia entre pantallas) | Prototipo descartable de una sola pantalla trivial, donde el costo de escribir Contract completo no se justifica | Garantiza estados consistentes por diseño y da una superficie de entrada única y testeable; a cambio, algo más de código ceremonial (Contract, reducer) incluso para pantallas simples |

## 5. Caso trampa

**"Mi pantalla solo tiene un `StateFlow<Boolean>` para un switch de configuración. ¿Igual necesito armar todo el Contract de MVI?"**

La respuesta obvia — "sí, todo el proyecto usa MVI, hay que ser consistente" — no es incorrecta como decisión de equipo, pero confunde la razón real por la que se elige MVI. MVI no se justifica por unificar la app a toda costa: se justifica cuando el **estado tiene combinaciones que pueden volverse inconsistentes**. Un switch aislado, sin loading ni error asociado, no tiene ese problema — un `StateFlow<Boolean>` suelto lo modela perfectamente bien y ningún estado "imposible" puede surgir de un solo booleano.

La trampa inversa también existe: pensar que como la pantalla es simple *hoy*, nunca va a necesitar MVI — y seis meses después esa misma pantalla del switch ahora tiene loading, error de guardado remoto, y un badge de "pendiente de sincronizar", momento en el que ya no hay forma barata de migrar sin tocar toda la UI que consumía los streams sueltos. La decisión real no es "¿mi equipo usa MVI en general?" sino "¿este estado específico puede volverse inconsistente si crece?" — y esa pregunta se vuelve a hacer cada vez que se le agrega una responsabilidad nueva a una pantalla que arrancó simple.

## 6. Conexión con arquitectura real

Timbax usa MVI en toda la capa Presentation, no como regla arbitraria sino porque casi ninguna pantalla real es tan trivial como el ejemplo del switch: `PlayersState` combina `isLoading`, `players` y `error` desde el día uno (ver `contract_state_event_effect.md`), y ese es justo el tipo de combinación que en MVVM clásico requeriría lógica condicional extra para evitar estados imposibles. El reducer conceptual `(State, Event) -> State` del ejemplo de esta sección es, en la práctica, lo que pasa dentro de cada rama del `when (event)` en `PlayersViewModel.onEvent()` (ver `viewmodel.md`) — no está extraído como función pura separada en Timbax hoy, pero es la lógica que ahí se testea indirectamente a través del ViewModel completo.