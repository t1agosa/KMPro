# 07_coroutines / `dispatchers.md`

---

## 1. Qué es

Un `Dispatcher` decide **en qué thread (o pool de threads) corre una coroutine**. La coroutine en sí es independiente del thread — el `Dispatcher` es quien la asigna a uno concreto, y puede reasignarla a otro distinto cada vez que se reanuda después de una suspensión.

Los tres principales:

- **`Dispatchers.Main`**: el thread de UI. Único thread donde está permitido tocar la UI directamente.
- **`Dispatchers.IO`**: pool de threads dimensionado para operaciones bloqueantes de red/disco — pensado para *esperar*, no para calcular.
- **`Dispatchers.Default`**: pool de threads dimensionado según los cores de CPU disponibles — pensado para cálculo pesado (parsing grande, ordenar listas enormes, procesamiento de imágenes).

## 2. El problema que resuelve

Sin `Dispatcher`, una coroutine simplemente corre en el thread donde fue lanzada. Si ese thread es el de UI (`Main`) y la coroutine hace algo bloqueante — una llamada de red síncrona, una query pesada a disco — la UI se congela: no se procesan touches, no se dibujan frames, y si tarda más de unos segundos el sistema operativo puede matar la app (ANR en Android).

El problema inverso también existe: usar un pool pensado para I/O (`IO`, con muchos threads porque la mayoría están *esperando*, no trabajando) para hacer cálculo pesado de CPU satura ese pool con trabajo que no es su propósito, y usar `Default` (pool acotado al número de cores) para I/O bloqueante puede dejar sin threads disponibles a otras tareas de cálculo real que sí los necesitan.

`Dispatchers` resuelve esto separando explícitamente "dónde correr según el tipo de carga", sin que el desarrollador tenga que crear y gestionar thread pools a mano.

## 3. Ejemplo mínimo comentado

```kotlin
class PlayerRepositoryImpl(
    private val api: PlayerApi,
    private val dao: PlayerDao
) : PlayerRepository {

    override suspend fun refreshPlayer(id: String): Player = withContext(Dispatchers.IO) {
        // acá adentro corre en el pool de IO: llamada de red bloqueante-friendly
        val remote = api.getPlayer(id)
        dao.save(remote.toEntity())
        remote.toDomain()
    }
}

class PlayerDetailViewModel(
    private val getPlayer: GetPlayerUseCase
) : ViewModel() {

    fun onEvent(event: PlayerDetailEvent.OnLoad) {
        viewModelScope.launch {
            // viewModelScope ya arranca en Dispatchers.Main.immediate
            _state.update { it.copy(isLoading = true) }
            val player = getPlayer(event.id) // adentro hace el switch a IO y vuelve solo
            // acá ya estamos de nuevo en Main, es seguro tocar el State para la UI
            _state.update { it.copy(player = player, isLoading = false) }
        }
    }
}
```

El punto clave: `withContext(Dispatchers.IO)` cambia el thread solo para ese bloque, y al terminar **vuelve automáticamente** al dispatcher que tenía la coroutine antes de entrar — no hace falta un `withContext(Dispatchers.Main)` explícito después para "volver". El `ViewModel` nunca tiene que preocuparse por dispatchers: la responsabilidad de decidir dónde correr el I/O vive en la capa `data`, no en `presentation`.

## 4. Matriz de criterio

| Escenario | Dispatcher | Por qué |
|---|---|---|
| Llamada de red (Ktor), query a DB (SQLDelight/Room), lectura de archivo | `Dispatchers.IO` | Pool grande pensado para threads que pasan la mayor parte del tiempo esperando, no consumiendo CPU |
| Parsear un JSON gigante, ordenar/filtrar una lista de miles de ítems, procesar una imagen | `Dispatchers.Default` | Pool acotado al número de cores, pensado para que la CPU esté realmente ocupada calculando |
| Actualizar el `State` del ViewModel, cualquier cosa que toque la UI | `Dispatchers.Main` | Es el único thread donde el framework de UI permite mutar vistas/composables de forma segura |
| Código dentro de `viewModelScope.launch { }` sin especificar dispatcher | ya es `Main` por default | `viewModelScope` usa `Dispatchers.Main.immediate` internamente, no hace falta declararlo |
| Una función de `data` que hace I/O | envolver el cuerpo en `withContext(Dispatchers.IO)` | La capa `data` es responsable de decidir su propio dispatcher — el `ViewModel` no debería saber ni importarle en qué thread corre el repository |

**NO uses:**
- `Dispatchers.IO` para cálculo pesado (ordenar una lista de 50.000 elementos, por ejemplo) — satura un pool dimensionado para *esperar*, no para *trabajar*, con tareas que si compiten por CPU real dejan sin threads disponibles a otras operaciones de I/O concurrentes.
- `Dispatchers.Main` para cualquier cosa que no sea actualizar UI o código muy liviano — cualquier bloqueo ahí congela la app entera.
- Un `Dispatcher` custom armado a mano sin necesidad real — `IO`/`Default`/`Main` cubren el 95% de los casos; un dispatcher custom (`limitedParallelism`, por ejemplo) es para casos puntuales de control fino de concurrencia, no el default.

## 5. Caso trampa

Un compañero escribe esto para "optimizar" la carga de la lista de jugadores, porque leyó que `Dispatchers.IO` "es para hacer cosas en paralelo, más rápido":

```kotlin
override suspend fun getPlayersSorted(): List<Player> = withContext(Dispatchers.IO) {
    val players = dao.getAll() // I/O real: query a la DB
    players.sortedByDescending { calculateComplexRanking(it) } // CPU-bound: cálculo pesado por cada jugador
}
```

Se ve razonable: todo el bloque está en `IO`, "así no bloquea la UI". La trampa es que `dao.getAll()` **sí** es I/O legítimo para `Dispatchers.IO`, pero `sortedByDescending { calculateComplexRanking(it) }` — si `calculateComplexRanking` hace matemática pesada por cada jugador (no solo comparar dos `Int`) — es trabajo de CPU, no de espera. Metido dentro del mismo `withContext(Dispatchers.IO)`, ese cálculo ocupa un thread del pool de IO que debería estar disponible para otras llamadas de red/disco concurrentes en la app, en vez de correr en el pool de `Default` que está pensado y dimensionado justamente para eso.

La versión correcta separa las dos responsabilidades:

```kotlin
override suspend fun getPlayersSorted(): List<Player> {
    val players = withContext(Dispatchers.IO) { dao.getAll() }
    return withContext(Dispatchers.Default) {
        players.sortedByDescending { calculateComplexRanking(it) }
    }
}
```

La pregunta trampa de entrevista es exactamente esta: *"¿esto está bien optimizado?"* — la respuesta no es "sí, porque usa `IO`", sino identificar que el bloque mezcla dos tipos de carga distintos bajo un solo dispatcher.

## 6. Conexión con arquitectura real

En Timbax, toda función de `RepositoryImpl` que toca `SQLDelight` o `GitLive Firebase SDK` envuelve su cuerpo en `withContext(Dispatchers.IO)` — esa decisión vive enteramente en la capa `data`, nunca se filtra hacia `domain` (los `UseCase` no importan `kotlinx.coroutines.Dispatchers` en absoluto) ni hacia `presentation` (el `ViewModel` solo hace `viewModelScope.launch { }` y confía en que quien implementa el repository ya resolvió en qué thread corre cada cosa). Esto es la misma Dependency Rule aplicada a concurrencia: la capa externa (`data`) sabe de detalles técnicos de ejecución: la interna (`domain`) no sabe que los dispatchers existen.