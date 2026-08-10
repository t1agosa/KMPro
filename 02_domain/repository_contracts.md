# repository_contracts.md

## 1. Qué es

El **Repository contract** es una `interface` de Kotlin que define *qué operaciones de datos existen*, sin decidir *cómo* se resuelven. Vive en `domain/repository/` y es pura firma — ningún detalle de Ktor, SQLDelight, Room o cualquier otra tecnología aparece ahí. La implementación real (`RepositoryImpl`) vive en la capa `data` y es la única que sabe de dónde vienen los datos.

```kotlin
interface PlayerRepository {
    fun getPlayers(): Flow<List<Player>>
    suspend fun updateScore(playerId: String, score: Int)
}
```

## 2. El problema que resuelve

Si el UseCase o el ViewModel dependieran directamente de `PlayerRepositoryImpl` (la clase concreta que usa SQLDelight), domain quedaría acoplado a una decisión de infraestructura. Cualquier cambio de tecnología de persistencia — migrar de SQLDelight a Room, agregar una fuente remota, cambiar la estrategia de cache — obligaría a tocar código en capas que no tienen nada que ver con esa decisión.

El contrato resuelve esto invirtiendo la dependencia: domain declara la interfaz que necesita, y es `data` quien depende de domain para implementarla (la aplica la Dependency Rule: las capas externas dependen de las internas, nunca al revés). Domain queda completamente ignorante de si los datos vienen de una API, una tabla local, o ambas combinadas.

## 3. Ejemplo mínimo comentado

```kotlin
// domain/repository/PlayerRepository.kt
// Solo el contrato: QUÉ se puede hacer, sin ninguna pista de CÓMO.
interface PlayerRepository {
    fun getPlayers(): Flow<List<Player>>
    suspend fun updateScore(playerId: String, score: Int)
}
```

La implementación vive del otro lado de la frontera, en `data`, y es la única que sabe qué tecnología hay detrás:

```kotlin
// data/repository/PlayerRepositoryImpl.kt
// Acá sí importa SQLDelight — pero domain nunca ve este archivo.
class PlayerRepositoryImpl(
    private val dao: PlayerDao
) : PlayerRepository {

    override fun getPlayers(): Flow<List<Player>> =
        dao.getAll().map { entities -> entities.map { it.toDomain() } }

    override suspend fun updateScore(playerId: String, score: Int) {
        dao.updateScore(playerId, score)
    }
}
```

El UseCase depende únicamente de la interfaz, nunca de la implementación:

```kotlin
// domain/usecases/GetPlayersUseCase.kt
class GetPlayersUseCase(private val repository: PlayerRepository) {
    operator fun invoke(): Flow<List<Player>> = repository.getPlayers()
}
```

## 4. Matriz de criterio

**Definir un método en el contrato cuando:**
- Representa una operación de datos real que el dominio necesita (`getPlayers`, `updateScore`), sin importar de dónde salga el dato.
- La firma expone tipos de domain (`Player`, `Flow<List<Player>>`) — nunca DTOs ni Entities, porque eso rompería la frontera que el contrato existe para proteger.

**NO poner en el contrato:**
- Detalles de implementación como parámetros de paginación específicos de una API (`cursor: String`) o de una query SQL — eso es responsabilidad de cómo `RepositoryImpl` decide resolver la operación puntual, no de qué operación existe.
- Métodos que combinan más de una fuente de datos con lógica propia (ej: "traer del cache si existe, si no ir a red") — esa orquestación vive dentro de `RepositoryImpl`, el contrato solo expone el resultado final (`getPlayers(): Flow<List<Player>>`), sin filtrar la estrategia usada para conseguirlo.

**Trade-off real:** con una sola implementación real por interfaz (como suele pasar en apps chicas o medianas), el contrato puede sentirse como una capa de indirección innecesaria — "¿para qué la interfaz si total solo hay un `PlayerRepositoryImpl`?". El valor no está en tener múltiples implementaciones simultáneas, sino en la testeabilidad: sin el contrato, testear un UseCase obligaría a instanciar (o mockear con herramientas pesadas) la implementación real con SQLDelight; con el contrato, se inyecta un `FakeRepository` liviano que cumple la interfaz sin tocar ninguna base de datos real.

## 5. Caso trampa

Un ViewModel necesita mostrar tanto la lista de jugadores como si el dispositivo tiene conexión a internet (para decidir si mostrar un banner de "modo offline"). La tentación es agregar `fun isOnline(): Boolean` directo a `PlayerRepository`, porque "ya está inyectado ahí y es rápido".

El problema es que la conectividad de red no es una operación sobre jugadores — es una capacidad de plataforma, no una responsabilidad del dominio de `Player`. Meterla en `PlayerRepository` mezcla dos contratos que no tienen relación conceptual, y el día que otra pantalla sin relación con jugadores necesite lo mismo, va a terminar inyectando `PlayerRepository` completo solo para leer `isOnline()`. La respuesta correcta es un contrato propio (`ConnectivityObserver` o similar, típicamente resuelto con `expect/actual` porque es una capacidad de plataforma), inyectado donde se necesite, sin forzarlo dentro de un repository que no tiene nada que ver semánticamente.

## 6. Conexión

En Timbax, `PlayerRepository` es el contrato que separa `GetPlayersUseCase` y `SaveScoreUseCase` de la decisión real de persistencia (hoy SQLDelight). Si mañana Timbax sumara sincronización en la nube para no perder partidas al cambiar de dispositivo, `PlayerRepositoryImpl` pasaría a orquestar SQLDelight (local) + Ktor (remoto) — pero ni el contrato en domain, ni los UseCases, ni el ViewModel necesitarían cambiar una sola línea, porque la firma (`Flow<List<Player>>`) sigue siendo exactamente la misma.