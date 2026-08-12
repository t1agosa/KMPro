# Null Safety

## 1. Qué es

Null safety es el sistema de tipos de Kotlin que distingue en tiempo de compilación entre tipos que pueden ser null (`String?`) y los que no (`String`), evitando el `NullPointerException` en el punto de uso — el compilador obliga a manejar el caso null antes de que el código compile.

## 2. El problema que resuelve

En lenguajes sin este sistema (Java clásico), cualquier referencia puede ser null en cualquier momento sin que el tipo lo indique, así que el `NullPointerException` puede aparecer en runtime en cualquier punto del código, sin aviso previo del compilador. El costo de esto se paga en producción, no en compile-time. Kotlin traslada ese chequeo al compilador: si un valor puede ser null, el tipo lo dice explícitamente (`String?`), y el compilador no te deja usarlo como si no lo fuera sin antes resolver el caso null.

## 3. Ejemplo mínimo comentado

```kotlin
var selectedPlayer: Player? = null

// safe call encadenado — devuelve null si selectedPlayer es null
val length = selectedPlayer?.name?.length
```

```kotlin
// elvis operator — valor por defecto si la izquierda es null
val displayName = selectedPlayer?.name ?: "Sin nombre"
```

```kotlin
// elvis para return temprano — patrón común al inicio de una función
fun savePlayer(player: Player?) {
    val validPlayer = player ?: return
    // acá validPlayer ya es Player no-nullable (smart cast)
}
```

```kotlin
// !! — evitar salvo casos muy puntuales y justificados
val name = selectedPlayer!!.name  // riesgo de NPE si selectedPlayer es null
```

## 4. Matriz de criterio

**Usar `?.` (safe call) cuando:** necesitás acceder a una propiedad/función de un valor nullable y está bien que el resultado también sea nullable si la cadena se corta.

**Usar `?:` (elvis) cuando:** tenés un valor por defecto razonable para el caso null, o necesitás cortar la ejecución temprano (`?: return`, `?: continue`, `?: throw ...`) para que el resto de la función trabaje con un tipo no-nullable garantizado.

**Usar `!!` cuando:** tenés una garantía de negocio real de que el valor no puede ser null en ese punto (ya fue validado antes en el mismo flujo) — y aun así, es preferible reestructurar el código para que el tipo sea no-nullable desde antes, en vez de usar `!!`.

**NO usar `!!` como atajo rápido:** es la trampa más común — usarlo "porque compila" reintroduce exactamente el riesgo de NPE que Kotlin fue diseñado para eliminar en compile-time.

**Trade-off real:** encadenar muchos `?.` seguidos (`a?.b?.c?.d`) es cómodo pero puede esconder una cadena de responsabilidades ambigua — si `b` nunca debería ser null según las reglas de negocio, es mejor modelar el tipo para que no lo sea, en vez de "tolerar" el null con `?.` en cada nivel.

## 5. Caso trampa

Usar `!!` "para que compile rápido" durante desarrollo y olvidarlo en el código final es el caso trampa más frecuente — parece inofensivo porque el código compila sin problema, pero traslada el crash de compile-time a runtime, justo lo opuesto al propósito de null safety:

```kotlin
// se ve inofensivo, compila, pasa el code review si nadie lo nota
fun getScore(player: Player?): Int = player!!.score
```

Si `player` llega null en producción (por ejemplo, por una condición de carrera o un estado inesperado), esto crashea la app exactamente igual que un NPE de Java — la diferencia es que acá fue una decisión explícita del desarrollador, no una limitación del lenguaje.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, el patrón `?: return` aparece típicamente al inicio de funciones de UseCase o ViewModel que reciben un id opcional o un resultado de búsqueda que puede no existir:

```kotlin
fun selectPlayer(playerId: String?) {
    val id = playerId ?: return
    // resto de la lógica trabaja con id no-nullable
}
```

El uso de `!!` se evita casi por completo en el código de producción — la única excepción real y documentada sería un valor que la propia lógica de negocio garantiza no-null en ese punto exacto (por ejemplo, después de un chequeo `require()` explícito), nunca como atajo genérico.