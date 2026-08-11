# 07_coroutines / `fundamentos_suspend_scope.md`

---

## 1. Qué es

Una coroutine es una tarea "suspendible": puede pausarse en medio de una operación lenta (red, disco, delay) y reanudarse después, sin bloquear el thread que la ejecuta. Mientras está pausada, ese thread queda libre para hacer otra cosa.

Dos piezas son indivisibles entre sí para que esto funcione:

- **`suspend fun`**: una función que puede pausarse y reanudarse. Solo se puede llamar desde otra `suspend fun` o desde dentro de una coroutine — el compilador necesita garantizar que el mecanismo de pausa/reanudación existe en ese punto del código.
- **`CoroutineScope`**: el contenedor que define el ciclo de vida dentro del cual viven las coroutines. Si el scope se cancela, todas sus coroutines hijas se cancelan en cascada.

Una `suspend fun` sin un scope que la lance no hace nada por sí sola — necesita alguien que la ejecute dentro de una coroutine real (`launch`, `async`, o ser llamada desde otra `suspend fun` que ya esté corriendo en un scope).

## 2. El problema que resuelve

Antes de coroutines, el código asíncrono en Kotlin/Java se resolvía con callbacks anidados o con threads manuales:

```kotlin
// Sin coroutines: callback hell
fun getPlayer(id: String, onResult: (Player) -> Unit, onError: (Exception) -> Unit) {
    api.getPlayer(id, object : Callback {
        override fun onSuccess(player: Player) {
            db.savePlayer(player, object : SaveCallback {
                override fun onSaved() = onResult(player)
                override fun onError(e: Exception) = onError(e)
            })
        }
        override fun onError(e: Exception) = onError(e)
    })
}
```

Cada paso asíncrono agrega un nivel de anidación, el manejo de errores se duplica en cada callback, y no hay forma clara de cancelar toda la cadena si el usuario sale de la pantalla a mitad de camino — hay que llevar banderas manuales (`isCancelled`) por todos lados.

Las coroutines resuelven esto con **código secuencial que se lee como síncrono, pero no bloquea el thread**, y con **structured concurrency**: toda coroutine vive dentro de un scope padre, así que cancelar el padre cancela automáticamente todo lo que lanzó, sin bookkeeping manual.

## 3. Ejemplo mínimo comentado

```kotlin
// domain — una operación suspendible, no sabe en qué thread corre ni quién la llama
suspend fun getPlayer(id: String): Player {
    return repository.getPlayerById(id) // otra suspend fun, la pausa se propaga
}

// presentation — el ViewModel es quien tiene un scope y decide lanzar la coroutine
class PlayerDetailViewModel(
    private val getPlayer: GetPlayerUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(PlayerDetailState())
    val state: StateFlow<PlayerDetailState> = _state.asStateFlow()

    fun onEvent(event: PlayerDetailEvent) {
        when (event) {
            is PlayerDetailEvent.OnLoad -> loadPlayer(event.id)
        }
    }

    private fun loadPlayer(id: String) {
        // viewModelScope es el CoroutineScope atado al ciclo de vida del ViewModel
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            val player = getPlayer(id) // se "pausa" acá sin bloquear el thread de UI
            _state.update { it.copy(player = player, isLoading = false) }
        }
    }
}
```

`getPlayer(id)` puede tardar 300ms hablando con la red, y durante ese tiempo el thread de UI queda libre para seguir dibujando frames — nada se bloquea, la coroutine simplemente se "pausa" ahí y el `ViewModel` retoma la ejecución exactamente donde quedó cuando la respuesta llega.

## 4. Matriz de criterio

| Escenario | Qué usar | Por qué |
|---|---|---|
| Una función de domain/data que hace I/O (red, DB) | `suspend fun` | Declara explícitamente que la operación puede tardar y necesita un contexto de coroutine para ejecutarse |
| Lanzar una operación desde el ViewModel | `viewModelScope.launch { }` | Ya trae `SupervisorJob() + Dispatchers.Main.immediate` y se cancela solo en `onCleared()` |
| Una función que solo hace cálculo sincrónico, sin I/O ni espera | función normal, **no** `suspend` | Marcarla `suspend` sin necesidad no aporta nada y confunde la intención de la firma |
| Necesitás lanzar una coroutine fuera de un ViewModel (ej: un `Worker`, un test) | crear un `CoroutineScope` propio explícito | No existe scope automático fuera de Android ViewModel/Compose; hay que declararlo y gestionarlo vos mismo (cancelarlo cuando corresponda) |
| Dos operaciones independientes que se pueden paralelizar | `async` dentro del scope (se ve en `launch_vs_async.md`) | `suspend fun` + `launch` a secas ejecuta todo secuencial aunque no dependa una de otra |

**NO uses:**
- `GlobalScope.launch { }` para "resolver rápido" un problema de scope — la coroutine vive tanto como la app, sin ciclo de vida, es la forma más común de generar una coroutine huérfana que sigue corriendo (y gastando batería/red) después de que la pantalla que la lanzó ya no existe.
- `suspend fun` en una función que un composable llama directamente en su cuerpo — el cuerpo de un `@Composable` no es un `CoroutineScope`; para eso están `LaunchedEffect` y compañía (visto en `09_ui_compose`).

## 5. Caso trampa

Tenés esta función en el repository:

```kotlin
class PlayerRepositoryImpl(
    private val api: PlayerApi,
    private val dao: PlayerDao
) : PlayerRepository {

    override suspend fun refreshPlayer(id: String) {
        val remote = api.getPlayer(id) // suspend fun
        dao.save(remote.toEntity())    // suspend fun
    }
}
```

Se ve inocente: dos `suspend fun` seguidas, todo bien. La trampa aparece cuando alguien la llama así desde el ViewModel:

```kotlin
fun onRefreshClicked() {
    viewModelScope.launch { repository.refreshPlayer(playerId) }
    viewModelScope.launch { repository.refreshPlayer(otherPlayerId) }
}
```

Esto **compila y funciona**, pero la pregunta trampa de entrevista es: *"¿estas dos llamadas corren en paralelo?"* La respuesta correcta es "depende, no necesariamente en el sentido que uno asume" — cada `launch` sí crea una coroutine independiente que puede intercalarse, pero eso no es lo mismo que garantizar paralelismo real de CPU/IO: sin especificar un `Dispatcher` (`Dispatchers.IO`, por ejemplo), ambas coroutines heredan el dispatcher del scope padre (`Dispatchers.Main` en `viewModelScope`), y aunque no bloquean el thread principal mientras esperan I/O, la ejecución del código no-suspendido entre pausas sigue siendo secuencial en ese mismo thread. El error conceptual común es pensar que "dos `launch`" automáticamente significa "dos threads reales trabajando a la vez" — no es así sin un dispatcher explícito pensado para eso.

## 6. Conexión con arquitectura real

En Timbax, cada `UseCase` expone su método de negocio como `suspend operator fun invoke(...)`, siguiendo exactamente el patrón de `getPlayer` de este archivo: el `UseCase` no sabe ni le importa qué thread lo ejecuta, solo declara que su operación puede suspenderse. Quien decide el scope y cuándo lanzar esa coroutine es siempre el `ViewModel`, nunca el `UseCase` ni el `Repository` — esto respeta la Dependency Rule (domain no depende de nada, ni siquiera de detalles de concurrencia de la capa de presentación) y es la base sobre la que se construye todo lo que viene en `dispatchers.md`, `launch_vs_async.md` y `supervisorjob_excepciones.md`.