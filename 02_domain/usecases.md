# usecases.md

## 1. Qué es

Un **UseCase** encapsula una sola acción de negocio en una clase con una única responsabilidad. Recibe lo que necesita (repositorios, otros datos), valida las reglas de negocio de esa acción puntual, y expone típicamente un solo método de entrada vía `operator fun invoke()`. Vive en `domain/usecases/` — una clase por acción, nunca una clase "manager" con métodos sueltos para todo.

```kotlin
class SaveScoreUseCase(private val repository: PlayerRepository) {
    suspend operator fun invoke(playerId: String, score: Int) {
        repository.updateScore(playerId, score)
    }
}
```

## 2. El problema que resuelve

Sin UseCases, la tentación es que el ViewModel llame directo al Repository y meta ahí mismo la lógica de negocio: validaciones, cálculos, reglas ("un score no puede ser negativo", "no se puede cerrar una ronda sin todos los jugadores puntuados"). El ViewModel termina mezclando dos responsabilidades que no deberían compartir clase: *cómo reflejar datos en el State* (su trabajo real) y *qué reglas de negocio aplican* (trabajo de domain).

Esto se nota cuando la misma regla de negocio hace falta en dos pantallas distintas — sin UseCase, la única forma de reusarla es copiar el código entre dos ViewModels, o crear un método suelto sin dueño claro. El UseCase le da a esa regla un lugar único donde vivir, testeable de forma aislada, sin necesidad de levantar ViewModel ni UI.

## 3. Ejemplo mínimo comentado

```kotlin
// domain/usecases/SaveScoreUseCase.kt
// Una sola acción de negocio: guardar el puntaje de un jugador, validando antes.
class SaveScoreUseCase(private val repository: PlayerRepository) {

    suspend operator fun invoke(playerId: String, score: Int): Result<Unit> {
        // Regla de negocio: acá vive la validación, no en el ViewModel.
        if (score < 0) {
            return Result.Error(IllegalArgumentException("Score no puede ser negativo"))
        }
        return try {
            repository.updateScore(playerId, score)
            Result.Success(Unit)
        } catch (e: Exception) {
            Result.Error(e)
        }
    }
}
```

Uso típico desde el ViewModel — nótese que `invoke()` permite llamarlo como si fuera una función:

```kotlin
// presentation/PlayersViewModel.kt
fun onEvent(event: PlayersEvent) {
    when (event) {
        is PlayersEvent.OnSaveScore -> viewModelScope.launch {
            when (val result = saveScoreUseCase(event.playerId, event.score)) {
                is Result.Success -> _state.update { it.copy(error = null) }
                is Result.Error -> _state.update { it.copy(error = result.exception.message) }
            }
        }
        else -> Unit
    }
}
```

## 4. Matriz de criterio

**Crear un UseCase cuando:**
- Hay una regla de negocio real que validar o aplicar (no solo "pasar el dato de acá para allá" sin ninguna lógica en el medio).
- La misma acción se necesita, o es previsible que se vaya a necesitar, desde más de un lugar (dos pantallas, un widget, una notificación).
- Se combinan datos de más de un Repository para producir un resultado (ej: `LoadDashboardUseCase` que junta `getPlayers()` y `getStats()` en paralelo).

**NO crear un UseCase cuando:**
- Es un simple passthrough sin ninguna regla: `fun invoke() = repository.getPlayers()` sin validación ni transformación agrega una capa de indirección que no aporta nada. Ahí el ViewModel puede llamar al Repository directo sin culpa.
- La "regla" es en realidad lógica de presentación (formatear un texto, decidir un color según el State) — eso es responsabilidad del ViewModel, no de domain.

**Trade-off real:** UseCases de un solo método (`invoke()`) multiplican la cantidad de archivos — por cada acción, una clase nueva. Es más ceremonia que tener un solo `PlayerInteractor` con varios métodos, pero a cambio cada UseCase es trivialmente testeable de forma aislada (un test, un archivo, sin necesidad de mockear 10 métodos de una clase gigante para testear uno solo).

## 5. Caso trampa

En Timbax, `GetPlayersUseCase` hoy es literalmente `repository.getPlayers()` sin ninguna validación — porque todavía no hay ninguna regla de negocio sobre *qué* jugadores mostrar. La pregunta trampa es: ¿vale la pena el UseCase si hoy no hace nada más que delegar?

La respuesta depende del criterio de "NO crear UseCase" de arriba: si es un passthrough puro y no hay evidencia de que vaya a dejar de serlo, no se justifica el archivo — el ViewModel puede llamar al Repository directo. Pero si ya se sabe que a corto plazo va a sumarse una regla real (por ejemplo, filtrar jugadores inactivos, u ordenar por score), ahí sí conviene crear el UseCase desde ahora, aunque hoy "no haga nada", porque el punto de extensión ya queda armado donde corresponde — y evita el problema inverso: descubrir la regla de negocio semanas después y tener que sacarla de un ViewModel que ya la mezcló con lógica de presentación.

## 6. Conexión

En Timbax, `SaveScoreUseCase` es el punto donde vive la validación real del juego (`score >= 0`, o reglas específicas de Chinchón sobre puntajes negativos permitidos en ciertas rondas) — el ViewModel solo decide cómo reflejar el `Result` en el `PlayersState`, nunca evalúa la regla él mismo. Es también el mismo caso que `GetPlayersUseCase` en el punto 5: el criterio de "crear el UseCase aunque hoy sea un passthrough" se sostiene mientras haya evidencia concreta de que una regla de negocio real (filtro, orden, validación) va a sumarse pronto — si no, se pospone hasta que aparezca esa evidencia.