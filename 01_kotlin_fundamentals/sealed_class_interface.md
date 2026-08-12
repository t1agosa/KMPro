# Sealed Class / Sealed Interface

## 1. Qué es

`sealed class` / `sealed interface` restringen una jerarquía de tipos a un conjunto cerrado y conocido en tiempo de compilación — todas las subclases están declaradas en el mismo módulo/paquete. Esto permite que un `when` sobre un sealed type sea exhaustivo: el compilador obliga a cubrir todos los casos posibles, sin necesitar un `else`.

```kotlin
sealed interface UiState {
    object Loading : UiState
    data class Success(val players: List<Player>) : UiState
    data class Error(val message: String) : UiState
}
```

## 2. El problema que resuelve

Sin un tipo cerrado, modelar "los posibles estados de una pantalla" con clases sueltas o un enum no le da al compilador ninguna garantía sobre qué implementaciones existen. Si mañana agregás un caso nuevo, nada te avisa qué `when` quedaron desactualizados — el bug aparece en runtime (un caso no contemplado que cae en un `else` genérico o ni siquiera se maneja) en vez de aparecer como error de compilación.

## 3. Ejemplo mínimo comentado

```kotlin
sealed interface UiState {
    object Loading : UiState
    data class Success(val players: List<Player>) : UiState
    data class Error(val message: String) : UiState
}

when (state) {
    is UiState.Loading -> showSpinner()
    is UiState.Success -> showList(state.players)
    is UiState.Error -> showError(state.message)
    // no hace falta "else" — el compilador sabe que estos son todos los casos posibles
}
```

## 4. Matriz de criterio

**Usar `sealed interface` cuando:** no necesitás estado común compartido en la superclase, y/o una clase necesita implementar esta jerarquía junto con otra (herencia múltiple de tipos, algo que `sealed class` no permite por ser clase). Es la opción preferida hoy por defecto.

**Usar `sealed class` cuando:** sí necesitás una propiedad o comportamiento común implementado en la superclase que todas las subclases heredan (por ejemplo, un `abstract val timestamp: Long` con lógica compartida).

**NO usar un `enum` cuando:** cada caso necesita llevar datos distintos — un enum no puede modelar eso, porque todos sus valores tienen la misma "forma" (mismas propiedades). `Error(message: String)` lleva un dato que `Loading` no tiene; eso es imposible de modelar limpio con enum.

**NO usar clases sueltas sin sellar cuando:** el conjunto de casos es finito y conocido de antemano (State, Event, Effect de MVI) — ahí perdés la exhaustividad del `when` y el chequeo del compilador.

**Trade-off real:** sealed types dan seguridad en compile-time pero requieren que todas las subclases estén en el mismo paquete/módulo (según la versión de Kotlin) — si necesitás que terceros externos extiendan la jerarquía libremente, sealed no es la herramienta correcta (ahí una interfaz abierta común tiene más sentido).

## 5. Caso trampa

Agregar un `else` "por las dudas" en un `when` exhaustivo sobre un sealed type parece inofensivo, pero elimina justamente la garantía que sealed te da:

```kotlin
when (state) {
    is UiState.Loading -> showSpinner()
    is UiState.Success -> showList(state.players)
    else -> {} // ⚠️ silenciosamente absorbe Error, y cualquier caso futuro
}
```

Si mañana se agrega `UiState.Empty`, el compilador ya no te avisa que falta manejarlo — cae directo en el `else` sin que nadie lo note. La forma correcta es cubrir cada caso explícitamente y dejar que el compilador marque error si falta uno.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el Contract de MVI (`State` / `Event` / `Effect`) usa `sealed interface` para `Event` y `Effect` en cada feature — por ejemplo, `PlayersEvent` con `OnPlayerClicked`, `OnRefresh`, etc. El `reduce`/`onEvent` del ViewModel es un `when` exhaustivo sobre ese sealed interface: si se agrega un nuevo `Event`, el compilador marca error en cada ViewModel que no lo contempla, evitando que una acción de usuario quede sin manejar silenciosamente.