# 07_coroutines / `launch_vs_async.md`

---

## 1. Qué es

`launch` y `async` son los dos "constructores" (builders) principales para crear una coroutine dentro de un `CoroutineScope`. Ambos arrancan una coroutine de forma inmediata (no esperan a que la pidas), pero difieren en qué devuelven y para qué se usa cada uno:

- **`launch`**: dispara una coroutine y no espera un resultado de vuelta (fire and forget). Devuelve un `Job`, que solo sirve para controlar el ciclo de vida de esa coroutine (cancelarla, esperar a que termine sin importar el valor).
- **`async`**: dispara una coroutine que **sí** produce un resultado. Devuelve un `Deferred<T>`, y el valor se obtiene llamando a `.await()` — que suspende hasta que el resultado esté listo.

## 2. El problema que resuelve

Sin esta distinción, cualquier operación asíncrona necesitaría un mecanismo propio para comunicar "terminé, acá está el resultado" (callbacks, listeners) o directamente no tendría forma de devolver nada de manera segura.

Pero el problema más concreto que resuelve `async` específicamente es el de **paralelizar operaciones independientes**. Si tenés dos llamadas de red que no dependen una de la otra, llamarlas con `suspend fun` normales una después de la otra las ejecuta secuencialmente — la segunda ni empieza hasta que la primera terminó, aunque no haya ninguna razón real para esperar:

```kotlin
// secuencial "disimulado": cada suspend fun espera a que termine la anterior
suspend fun loadDashboard(): Dashboard {
    val players = repository.getPlayers() // tarda 300ms
    val stats = repository.getStats()     // tarda 300ms, arranca recién cuando termina la de arriba
    return Dashboard(players, stats)      // total: ~600ms
}
```

`async` resuelve esto lanzando ambas coroutines **ya**, en paralelo, y solo suspendiendo cuando de verdad necesitás el valor de cada una.

## 3. Ejemplo mínimo comentado

```kotlin
// launch: no necesito el resultado, solo que la acción ocurra
fun onSaveScoreClicked(playerId: String, newScore: Int) {
    viewModelScope.launch {
        saveScoreUseCase(playerId, newScore) // fire and forget, no me interesa un valor de retorno
    }
}

// async: necesito combinar dos resultados independientes
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val players = async { repository.getPlayers() } // arranca YA, en paralelo
    val stats = async { repository.getStats() }      // arranca YA, en paralelo
    Dashboard(players.await(), stats.await())         // espera ambas, total: ~300ms
}
```

`coroutineScope { }` acá crea un scope hijo que espera a que **todos** sus `async` terminen antes de devolver el control — es la forma correcta de agrupar varios `async` relacionados sin depender del scope externo (`viewModelScope`) para esa espera.

## 4. Matriz de criterio

| Escenario | Qué usar | Por qué |
|---|---|---|
| Guardar un score, actualizar un log, disparar analytics | `launch` | No hay un valor que necesites de vuelta, solo que la acción se ejecute |
| Cargar `players` y `stats` para armar un dashboard, dos consultas independientes | `async` + `await()` en ambas | Paraleliza operaciones que no dependen entre sí, reduce el tiempo total de espera |
| Cargar `player` y después, con ese `player.id`, cargar sus `stats` | `suspend fun` secuenciales (ni `launch` ni `async`) | Hay dependencia real entre los dos pasos — paralelizar no tiene sentido si el segundo necesita el resultado del primero |
| Necesitás lanzar una coroutine desde el `ViewModel` en respuesta a un `Event` | `launch` | El `ViewModel` no "espera" nada del `Event` — reacciona y actualiza `State`, no retorna un valor a quien llamó `onEvent` |
| Ejecutar dos `async` pero llamar `.await()` de a uno, con lógica en el medio antes del segundo `await` | revisar el diseño — probablemente conviene ordenar los `await()` juntos al final | Si el `.await()` del primero bloquea el punto en el que se **lanza** el segundo `async`, ya perdiste el paralelismo aunque técnicamente uses `async` |

**NO uses:**
- `async { }` seguido inmediatamente de `.await()` en la misma línea, sin otro `async` en paralelo — eso es exactamente lo mismo que llamar la `suspend fun` directo, pero con más código y la sobrecarga conceptual de manejar un `Deferred` sin necesidad.
- `launch` cuando necesitás el resultado de la operación para seguir el flujo — `launch` no te da forma limpia de extraer un valor (tendrías que usar una variable externa mutable capturada por el lambda, lo cual es un anti-patrón de concurrencia).

## 5. Caso trampa

Un compañero escribe esto pensando que está paralelizando la carga del dashboard:

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val players = async { repository.getPlayers() }.await() // await() inmediato
    val stats = async { repository.getStats() }.await()     // await() inmediato
    Dashboard(players, stats)
}
```

Compila perfecto, usa `async`, y sin embargo **no hay ningún paralelismo real** — es exactamente tan secuencial como llamar a las dos `suspend fun` directo, sin ningún `async` de por medio. El motivo: `.await()` se llama inmediatamente después de crear cada `async`, así que la coroutine se suspende ahí mismo esperando el resultado del primer `async` **antes** de que el segundo `async { repository.getStats() }` siquiera se cree y arranque.

La versión que sí paraleliza es la del punto 3: **primero** se crean ambos `async` (ambos arrancan ya), y **recién después** se llaman los dos `.await()`. El orden en el que declarás el `async` respecto al `.await()` es lo que determina si hay paralelismo real o no — la sola presencia de la palabra `async` en el código no lo garantiza. Esta es la pregunta trampa clásica: *"¿este código está paralelizado?"* — hay que leer con cuidado dónde está cada `.await()`, no asumir por ver `async` en pantalla.

## 6. Conexión con arquitectura real

En Timbax, la pantalla de resumen de partida (`GameSummaryScreen`) necesita combinar los scores finales de todos los jugadores con las estadísticas históricas de cada uno — dos consultas independientes a `PlayerRepository` y `StatsRepository` respectivamente. El `UseCase` que arma ese resumen (`GetGameSummaryUseCase`) usa `coroutineScope { }` con dos `async` siguiendo exactamente el patrón del punto 3, en vez de dos llamadas `suspend fun` secuenciales — la diferencia se nota en el tiempo de carga real de esa pantalla cuando hay latencia de red. El resto de los `Event` del `ViewModel` (guardar un score, marcar una ronda como jugada) usan `launch` simple, porque son fire-and-forget que solo necesitan actualizar el `State` cuando terminan, no combinar resultados de operaciones paralelas.