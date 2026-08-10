# RepositoryImpl

## 1. Qué es

`RepositoryImpl` es la clase concreta que vive en la capa `data` e implementa una interfaz `Repository` definida en `domain`. Es donde efectivamente se decide **de dónde** sale un dato (¿de la red? ¿de la base local? ¿de una cloud DB como Firebase? ¿de varias combinadas?) y **qué hacer** cuando esas fuentes no están de acuerdo entre sí. El domain solo sabe que existe un `PlayerRepository` con un método `getPlayers(): Flow<List<Player>>` — no sabe, ni le importa, si por debajo hay SQLDelight, Ktor, Firebase, o las tres cosas orquestadas.

```kotlin
class PlayerRepositoryImpl(
    private val dao: PlayerDao,
    private val api: PlayerApi
) : PlayerRepository {

    override fun getPlayers(): Flow<List<Player>> =
        dao.getAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun refreshPlayers() {
        val remotePlayers = api.fetchPlayers()
        dao.insertAll(remotePlayers.map { it.toEntity() })
    }
}
```

## 2. El problema que resuelve

Sin este patrón, cada `UseCase` (o peor, cada `ViewModel`) tendría que saber directamente cómo hablar con Ktor, con SQLDelight y con Firebase, decidiendo caso por caso si conviene pedir a la red, leer de la DB local, o sincronizar con la nube. Eso rompe la Dependency Rule dos veces: `presentation` terminaría dependiendo de detalles de infraestructura (`data`), y esa lógica de "de dónde saco el dato" quedaría duplicada y dispersa en vez de vivir en un solo lugar. `RepositoryImpl` centraliza esa decisión una sola vez, detrás de un contrato estable que el resto de la app no necesita reabrir nunca.

## 3. Ejemplo mínimo comentado

Estrategia **local-first** (la más común para apps como Timbax, donde la fuente de verdad real vive en la DB local y la red/cloud solo la alimentan):

```kotlin
class PlayerRepositoryImpl(
    private val dao: PlayerDao,
    private val api: PlayerApi
) : PlayerRepository {

    // La UI siempre observa la DB local — nunca la red directamente.
    override fun getPlayers(): Flow<List<Player>> =
        dao.getAll().map { entities -> entities.map { it.toDomain() } }

    // La red solo se usa para "alimentar" la DB, nunca para responder directo a la UI.
    override suspend fun refreshPlayers() {
        val remoteDtos = api.fetchPlayers()
        dao.insertAll(remoteDtos.map { it.toEntity() }) // dao.getAll() emite el update solo
    }

    override suspend fun savePlayer(player: Player) {
        dao.insert(player.toEntity())
        // acá podría dispararse también un push a Firebase, si Timbax sincroniza en la nube
    }
}
```

La clave: `getPlayers()` nunca le pide nada a `api` directamente. `refreshPlayers()` escribe en `dao`, y como `dao.getAll()` es un `Flow` reactivo (SQLDelight/Room emiten automáticamente en cada cambio de tabla), la UI se entera del dato nuevo sin que `RepositoryImpl` tenga que "empujar" nada manualmente.

### Caso con tres fuentes: local + remote API + Firebase (GitLive SDK)

Cuando además hay sincronización en la nube (como en Timbax, con Firestore vía GitLive), `RepositoryImpl` termina orquestando tres fuentes en vez de dos, pero **la regla de single source of truth no cambia**: la UI sigue leyendo solo de `dao`, y tanto la API REST como Firebase son, para la UI, invisibles — solo alimentan la DB local.

```kotlin
class PlayerRepositoryImpl(
    private val dao: PlayerDao,
    private val firestore: FirebaseFirestore // GitLive SDK, KMP-compatible
) : PlayerRepository {

    // La UI sigue observando únicamente la DB local.
    override fun getPlayers(): Flow<List<Player>> =
        dao.getAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun savePlayer(player: Player) {
        // 1. Se escribe primero en local — la UI reacciona al instante, sin esperar la nube.
        dao.insert(player.toEntity())

        // 2. Se sincroniza con Firestore en segundo plano.
        //    Si esto falla (sin conexión), el dato local ya está guardado y consistente;
        //    la sync puede reintentarse después sin bloquear al usuario.
        firestore.collection("players")
            .document(player.id)
            .set(player.toFirestoreDto())
    }

    // Observa cambios remotos en Firestore y los vuelca a la DB local —
    // nunca directo a la UI, siempre pasando por dao primero.
    suspend fun observeRemoteChanges() {
        firestore.collection("players")
            .snapshots
            .collect { snapshot ->
                val remotePlayers = snapshot.documents.map { it.data<PlayerFirestoreDto>() }
                dao.insertAll(remotePlayers.map { it.toEntity() })
            }
    }
}
```

Notar el detalle de `GitLive`: es un SDK pensado desde el día uno para KMP (a diferencia del SDK oficial de Firebase, que es Android/iOS nativo por separado) — expone una API común en `commonMain` que internamente delega a cada plataforma, similar en espíritu a lo que hace Ktor con sus engines por plataforma.

## 4. Matriz de criterio

**Local-first (single source of truth = DB local):**
- Usar cuando: la app necesita funcionar offline, o cuando querés que la UI reaccione instantáneamente a cambios sin esperar la red (que es exactamente el caso de Timbax: los puntajes se guardan localmente sí o sí, la sync a Firebase es secundaria).
- NO usar cuando: el dato tiene que ser siempre el más fresco posible y no tolera ni un segundo de desincronización (ej: precio de una acción en bolsa).
- Trade-off: agregás una capa de indirección (escribís en DB, después la DB te devuelve el dato) en vez de devolver la respuesta de red directo — a cambio, ganás una sola fuente de verdad consistente y funcionamiento offline gratis.

**Network-first (la red responde directo, la DB es solo caché opcional):**
- Usar cuando: el dato cambia todo el tiempo y no tiene sentido mostrar algo desactualizado (ej: resultado de un partido en vivo).
- NO usar cuando: necesitás que la app funcione sin conexión, o cuando el costo de una llamada de red repetida es alto.
- Trade-off: UI más "fresca", pero perdés funcionamiento offline y le agregás latencia de red a cada lectura.

**Combinado (cache-first con fallback, "mostrar cache mientras se refresca"):**
- Usar cuando: querés lo mejor de los dos mundos — mostrar algo inmediato (cache) mientras en paralelo se dispara un refresh silencioso.
- NO usar cuando: el equipo es chico y esta complejidad extra (manejar el estado de "mostrando cache viejo" vs "ya refrescado") no se justifica todavía para el tamaño del proyecto.
- Trade-off: mejor UX percibida, pero más superficie de bugs (¿qué pasa si el refresh falla? ¿la UI debería avisar que está mostrando datos viejos?).

**Sync con Firebase/GitLive como fuente adicional (local + cloud):**
- Usar cuando: necesitás que el dato sobreviva a un cambio de dispositivo o se comparta entre usuarios (ej: un jugador que abre Timbax en dos celulares), y la app ya tiene una DB local como fuente de verdad primaria.
- NO usar cuando: el dato es puramente local a la sesión/dispositivo y no hay necesidad real de multi-dispositivo o backend compartido — sumar Firebase ahí es complejidad sin beneficio.
- Trade-off: ganás persistencia multi-dispositivo y tiempo real, pero sumás una fuente más para orquestar (¿qué pasa si local y remoto divergen? ¿quién gana, last-write-wins u otra estrategia?), y quedás atado a la disponibilidad/latencia de Firestore para la parte de sync.

## 5. Caso trampa

Un repository "network-first" mal armado, que parece funcionar bien en desarrollo pero esconde un problema:

```kotlin
// ❌ trampa: la UI recibe la respuesta de la red directo, sin pasar por la DB
override fun getPlayers(): Flow<List<Player>> = flow {
    val remotePlayers = api.fetchPlayers()
    dao.insertAll(remotePlayers.map { it.toEntity() }) // guarda en DB...
    emit(remotePlayers.map { it.toDomain() })          // ...pero emite el DTO de red, no lo que quedó guardado
}
```

El bug no es obvio a simple vista: guarda en la DB (parece "correcto"), y emite el resultado (la UI se actualiza). El problema aparece cuando la DB tiene alguna lógica propia entre medio — un `INSERT OR REPLACE` que descarta un campo, un trigger, un mapper con una regla de negocio distinta al mapper de red. En ese caso, lo que la UI muestra (`remotePlayers.map { it.toDomain() }`) **no es lo mismo** que lo que efectivamente quedó persistido en la DB. Si el usuario cierra la app y la vuelve a abrir, va a ver algo distinto a lo que vio un segundo antes de cerrarla — porque ahora sí está leyendo de la DB (vía `dao.getAll()` en el arranque), y esa nunca fue la misma fuente que alimentó la emisión anterior.

El mismo error, versión Firebase: guardar en Firestore y emitir directo el dato que se mandó a la nube, en lugar de esperar a que `observeRemoteChanges()` lo vuelque a `dao` y sea ese el que emita. Si el `set()` a Firestore falla silenciosamente (por ejemplo, sin conexión), la UI ya mostró un dato que nunca llegó a persistirse en ningún lado confiable.

La solución es la regla de "single source of truth" ya mencionada en la matriz: si la DB local es la fuente de verdad, **todo** lo que la UI ve tiene que pasar por ella, sin atajos — ni siquiera para "ahorrarse" una vuelta extra de lectura.

## 6. Conexión

En Timbax, `PlayerRepositoryImpl` es exactamente este patrón: implementa la interfaz `PlayerRepository` de `02_domain`, y decide puertas adentro si el dato sale de SQLDelight local, si además sincroniza con Firestore vía GitLive, o ambas cosas orquestadas. El `SaveScoreUseCase` (ya documentado en `usecases.md`) nunca sabe nada de esto — solo le llama a `repository.savePlayer(player)` y confía en que, sea cual sea la estrategia interna (local-only, local + cloud sync), el contrato se cumple. Esta es la Dependency Rule en acción tal cual está descrita en el machete: `data` depende de `domain` (implementa su interfaz), nunca al revés.