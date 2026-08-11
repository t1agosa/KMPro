# Flow básico

## 1. Qué es

`Flow<T>` es un stream asincrónico de valores que se emiten en el tiempo (0, 1 o muchos). Es **frío** (cold): el bloque productor no corre hasta que alguien lo colecta con `.collect { }`, y cada colector dispara su propia ejecución del productor, de forma completamente independiente.

## 2. El problema que resuelve

Antes de Flow, observar un dato que cambia con el tiempo (una tabla de DB, una preferencia) significaba elegir entre polling manual (consultar cada X segundos, ineficiente) o callbacks/listeners escritos a mano (propensos a leaks si no se desuscriben bien). Flow da un mecanismo estándar, integrado con coroutines y `suspend`, para modelar "esto puede cambiar, avisame cuando pase" sin ninguna de esas dos soluciones artesanales.

## 3. Ejemplo mínimo comentado

```kotlin
interface PlayerRepository {
    fun getPlayers(): Flow<List<Player>>
}

class PlayerRepositoryImpl(private val dao: PlayerDao) : PlayerRepository {
    override fun getPlayers(): Flow<List<Player>> =
        dao.getAll() // Flow que emite cada vez que cambia la tabla
            .map { entities -> entities.map { it.toDomain() } } // DTO/Entity -> Model
}
```

```kotlin
// consumo típico dentro de un ViewModel
viewModelScope.launch {
    playerRepository.getPlayers().collect { players ->
        _state.update { it.copy(players = players) }
    }
}
```

## 4. Matriz de criterio

| Usar cuando | NO usar cuando |
|---|---|
| Necesitás observar un dato que cambia en el tiempo (tabla de DB, preferencia reactiva) | Necesitás un solo valor puntual, de una sola vez (ahí alcanza con una `suspend fun` que devuelve `T` directo) |
| El dato tiene múltiples emisiones futuras relevantes | La operación es un comando one-shot (guardar, eliminar) sin necesidad de stream de respuesta |

**Trade-off real:** al ser frío, si dos partes de la UI colectan el mismo `Flow` de forma independiente (por ejemplo, dos pantallas llamando `getPlayers()` sin compartir instancia), el productor corre dos veces — dos queries a la DB, dos llamadas de red si el Flow envuelve una. Esto se resuelve compartiendo el Flow como `StateFlow` (vía `.stateIn()`) cuando hay múltiples colectores reales, no dejando que cada uno dispare su propia ejecución.

## 5. Caso trampa

Pensás que como ya llamaste a `repository.getPlayers()` una vez arriba en el ViewModel y coleccionaste el resultado, si otro composable "escucha" el mismo Flow más tarde va a recibir el mismo dato sin volver a consultar la DB. Falso: cada `.collect{}` es una ejecución nueva del bloque productor. Si `getPlayers()` internamente hiciera algo costoso (no solo `dao.getAll()`, sino por ejemplo una llamada de red antes de mapear), estarías disparando esa llamada costosa una vez por cada colector, sin darte cuenta.

## 6. Conexión con arquitectura real

En Timbax, `PlayerRepositoryImpl.getPlayers()` (capa `data`, package `03_data`) devuelve `Flow<List<Player>>` mapeando desde el DAO. El contrato `PlayerRepository.getPlayers(): Flow<List<Player>>` vive en `domain` (`02_domain/repository_contracts.md`) — domain solo conoce el tipo `Flow`, nunca sabe si detrás hay SQLDelight, Room o una API. Este Flow termina siendo la fuente que el ViewModel colecta para actualizar el `State` (ver `06_presentation_mvi`).