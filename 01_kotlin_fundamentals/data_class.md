# Data Class

## 1. Qué es

Una `data class` genera automáticamente `equals()`/`hashCode()` (comparación por valor de sus propiedades, no por referencia), `toString()` legible, y `copy()` (crear una copia modificando solo algunos campos).

```kotlin
data class Player(val id: String, val name: String, val score: Int)
```

## 2. El problema que resuelve

Sin `data class`, cada clase que solo transporta datos obligaría a escribir a mano `equals()`, `hashCode()` y `toString()` — código repetitivo, propenso a errores (olvidar actualizar `equals()` cuando agregás una propiedad nueva), y sin ninguna forma cómoda de crear una copia parcial de una instancia inmutable sin reescribir todos sus campos uno por uno.

## 3. Ejemplo mínimo comentado

```kotlin
data class Player(val id: String, val name: String, val score: Int)

val updated = player.copy(score = player.score + 10)  // nueva instancia, mismo id/name
```

```kotlin
// equals por valor, no por referencia
val a = Player("1", "Tiago", 10)
val b = Player("1", "Tiago", 10)
println(a == b)   // true — compara valores, no identidad de objeto
println(a === b)  // false — son instancias distintas en memoria
```

## 4. Matriz de criterio

**Usar `data class` cuando:** la clase existe principalmente para transportar datos, y la igualdad por valor tiene sentido semántico (dos instancias con los mismos campos representan "lo mismo").

**NO usar `data class` cuando:** la igualdad por valor no tiene sentido en el dominio — por ejemplo, una clase que representa una conexión activa a un socket, donde dos conexiones con los mismos parámetros no son "la misma conexión" (ahí importa la identidad, no el valor).

**NO usar `data class` cuando:** la clase tiene lógica de comportamiento compleja más que solo transportar datos — una clase regular comunica mejor la intención de que el foco está en el comportamiento, no en los datos.

**Trade-off real:** `copy()` es cómodo pero solo hace shallow copy — si una propiedad es un objeto mutable, `copy()` no lo clona, solo copia la referencia. Esto puede generar bugs sutiles si asumís inmutabilidad total sin verificar que todas las propiedades anidadas también sean inmutables.

## 5. Caso trampa

Asumir que `copy()` protege completamente contra mutación compartida cuando alguna propiedad es una colección mutable:

```kotlin
data class Team(val name: String, val players: MutableList<Player>)

val teamA = Team("Rojo", mutableListOf(player1))
val teamB = teamA.copy(name = "Azul")

teamB.players.add(player2)
// teamA.players TAMBIÉN cambia — copy() copió la REFERENCIA a la misma MutableList
```

La forma correcta es usar `List` inmutable (no `MutableList`) en el `data class`, o clonar explícitamente la colección dentro de `copy()`.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el `State` de cada Contract MVI es siempre una `data class` — es precisamente lo que permite `_state.update { it.copy(isLoading = false) }` para producir un nuevo estado inmutable a partir del anterior sin reescribir todos los campos. Además, `equals()` por valor es lo que le permite a Compose comparar si el `State` realmente cambió entre una recomposición y otra, y decidir si vale la pena recomponer o saltear (skip).