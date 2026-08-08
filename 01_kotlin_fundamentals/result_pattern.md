# result_pattern.md

## 1. Qué es

El **Result Pattern** es un tipo envoltorio (`sealed interface`) que representa el resultado de una operación que puede fallar, sin recurrir a excepciones crudas propagándose sin control. Vive en `domain/` (o `domain/util/`) y tiene dos ramas cerradas: éxito con un dato, o error con una excepción.

```kotlin
sealed interface Result<out T> {
    data class Success<T>(val data: T) : Result<T>
    data class Error(val exception: Throwable) : Result<Nothing>
}
```

## 2. El problema que resuelve

Sin un tipo explícito para el resultado, una función que puede fallar tiene dos caminos típicos, y ambos son problemáticos. El primero es lanzar excepciones directo: `suspend fun getPlayers(): List<Player>` que puede tirar `IOException` en cualquier punto — el problema es que la firma no lo dice, así que quien la llama no tiene ninguna garantía en tiempo de compilación de que debe manejar el error; se entera recién en runtime, cuando la excepción llega sin `catch`. El segundo camino es devolver `null` en caso de error (`fun getPlayers(): List<Player>?`) — pero eso pierde información: un `null` no distingue "no hay jugadores" de "falló la conexión a la API", dos situaciones que la UI necesita mostrar de forma completamente distinta.

El Result Pattern resuelve ambos problemas a la vez: la firma de la función deja explícito que puede fallar (`Result<List<Player>>`), y el `sealed interface` obliga —vía `when` exhaustivo— a manejar ambos casos en el punto de consumo, sin poder ignorar el error por accidente.

## 3. Ejemplo mínimo comentado

```kotlin
// domain/util/Result.kt
// out T: covariante porque Result solo PRODUCE T, nunca lo recibe como parámetro.
sealed interface Result<out T> {
    data class Success<T>(val data: T) : Result<T>
    data class Error(val exception: Throwable) : Result<Nothing>
}
```

Uso típico en un UseCase, envolviendo una operación que puede fallar:

```kotlin
// domain/usecases/SaveScoreUseCase.kt
class SaveScoreUseCase(private val repository: PlayerRepository) {
    suspend operator fun invoke(playerId: String, score: Int): Result<Unit> {
        return try {
            repository.updateScore(playerId, score)
            Result.Success(Unit)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}
```

Consumo exhaustivo desde el ViewModel — el compilador exige cubrir ambos casos:

```kotlin
viewModelScope.launch {
    when (val result = saveScoreUseCase(playerId, score)) {
        is Result.Success -> _state.update { it.copy(error = null) }
        is Result.Error -> _state.update { it.copy(error = result.exception.message) }
    }
}
```

## 4. Matriz de criterio

**Usar Result cuando:**
- La operación puede fallar de forma esperable (red, disco, validación de negocio) y quien la consume necesita distinguir explícitamente éxito de error para reaccionar distinto en cada caso (mostrar datos vs. mostrar mensaje de error).
- Se quiere que el compilador fuerce el manejo del error en el punto de consumo, en vez de confiar en que alguien recuerde poner un `try/catch`.

**NO usar Result cuando:**
- La función no puede fallar de ninguna forma relevante para el dominio (una transformación pura, un cálculo matemático simple) — envolver en `Result` algo que nunca falla es ceremonia sin propósito.
- El estado de carga (`isLoading`) — eso nunca va dentro de `Result`, porque `Loading` no es el resultado de una operación puntual, es estado de presentation que vive en el `State` del ViewModel. Meterlo en `Result` (`Result.Loading`) mezcla dos responsabilidades distintas: el resultado de una llamada finalizada vs. el estado transitorio mientras esa llamada todavía no terminó.

**Trade-off real:** frente a lanzar excepciones directo, `Result` agrega un nivel de indirección (hay que desenvolver el tipo con `when` o funciones de extensión tipo `.getOrNull()`) en cada punto de consumo. A cambio, elimina una categoría entera de bugs de "excepción no capturada que crashea la app", porque el compilador no deja ignorar el caso de error silenciosamente — el costo es sintáctico, la ganancia es en tiempo de compilación.

## 5. Caso trampa

Un `Result.Error` envuelve un `CancellationException` porque la corrutina que hacía la llamada de red fue cancelada (por ejemplo, el usuario navegó fuera de la pantalla mientras la request estaba en vuelo). El `catch (e: Exception)` genérico de un UseCase la captura igual que cualquier otra excepción, la envuelve en `Result.Error(e)`, y el ViewModel termina mostrando "Error: StandaloneCoroutine was cancelled" como si fuera un fallo real de red.

El problema de fondo es más grave que un mensaje feo: `CancellationException` es el mecanismo que usa `structured concurrency` para propagar la cancelación hacia las corrutinas hijas — si se captura y no se relanza, se rompe la cancelación en cascada del scope completo. La regla es: nunca atrapar `CancellationException` de forma silenciosa. El `catch` debe verificar el tipo y relanzarla:

```kotlin
} catch (e: CancellationException) {
    throw e // nunca capturar la cancelación — hay que dejarla propagarse
} catch (e: Exception) {
    Result.Error(e)
}
```

## 6. Conexión

En Timbax, todos los UseCases que tocan `PlayerRepository` (`SaveScoreUseCase`, y cualquier otro que involucre I/O real) devuelven `Result<T>`, y el `PlayersViewModel` nunca recibe una excepción cruda — siempre resuelve un `when` exhaustivo sobre `Success`/`Error` antes de actualizar el `PlayersState`. Esto conecta directo con `model.md`: así como `Player` es el tipo que atraviesa las capas sin filtrar detalles de infraestructura, `Result<Player>` es el tipo que atraviesa las capas sin filtrar el tipo de excepción concreto (`SQLiteException`, `IOException`) — domain y presentation solo conocen `Result.Error(exception: Throwable)`, genérico y desacoplado de la tecnología que originó el fallo.