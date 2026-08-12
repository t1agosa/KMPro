# Extension Functions

## 1. Qué es

Las extension functions agregan una función a una clase existente sin heredar de ella ni modificar su código fuente. Es el mecanismo detrás de los Mappers de Clean Architecture (`RemoteModel.toDomain()`).

```kotlin
fun PlayerDto.toDomain(): Player = Player(id = this.id, name = this.name, score = this.score)
```

## 2. El problema que resuelve

Sin extension functions, agregar comportamiento a un tipo que no controlás (una clase de una librería externa, o un tipo de la stdlib como `String`) obliga a envolver el tipo en un wrapper propio o a escribir funciones utilitarias sueltas del estilo `PlayerMapper.toDomain(dto)` — que rompen la legibilidad fluida de "el objeto hace algo" y dispersan la lógica en clases utilitarias genéricas en vez de vivir cerca del tipo al que conceptualmente pertenecen.

## 3. Ejemplo mínimo comentado

```kotlin
fun PlayerDto.toDomain(): Player = Player(id = this.id, name = this.name, score = this.score)
```

```kotlin
// extension function sobre un tipo de la stdlib
fun String.isValidPlayerName(): Boolean = this.isNotBlank() && this.length <= 20

val name = "Tiago"
if (name.isValidPlayerName()) { /* ... */ }
```

## 4. Matriz de criterio

**Usar extension functions cuando:** necesitás agregar comportamiento de utilidad a un tipo que no controlás (tipos de librerías externas, tipos de la stdlib de Kotlin) sin ensuciarlo con herencia innecesaria.

**Usar extension functions cuando:** estás mapeando entre capas de Clean Architecture (DTO/Entity → Domain Model) — es el lugar natural para vivir de los Mappers, manteniendo la conversión cerca del tipo origen sin acoplar el modelo de dominio a la tecnología de origen.

**NO usar extension functions cuando:** necesitás acceder a miembros `private` de la clase que "extendés" — por debajo, el compilador la traduce a una función estática que recibe el receptor como parámetro, así que no tiene acceso real a los internals de la clase (no modifica realmente la clase original, es azúcar sintáctico).

**NO usar extension functions cuando:** el comportamiento es específico del dominio de negocio y debería vivir como método real de una clase que sí controlás — usarlas para todo, incluso donde tiene más sentido un método de instancia normal, dispersa la lógica y dificulta encontrarla.

**Trade-off real:** ganás legibilidad fluida y separación de responsabilidades (mappers fuera del modelo de dominio), pero perdés polimorfismo — una extension function no puede ser `override` en una subclase, se resuelve estáticamente según el tipo declarado de la variable en compile-time, no el tipo real en runtime.

## 5. Caso trampa

Asumir que una extension function se resuelve polimórficamente igual que un método de instancia:

```kotlin
open class Player(val name: String)
class VipPlayer(name: String) : Player(name)

fun Player.describe() = "Player regular"
fun VipPlayer.describe() = "Player VIP"

val p: Player = VipPlayer("Tiago")
println(p.describe())  // imprime "Player regular" — se resuelve por el tipo DECLARADO, no el real
```

Esto sorprende a quien viene de pensar en extension functions como métodos "normales" — se resuelven estáticamente según el tipo de la variable en el punto de compilación, no dinámicamente según la instancia real en runtime.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, todos los Mappers de la capa `data` son extension functions: `PlayerDto.toDomain()`, `PlayerEntity.toDomain()`, etc. Esto mantiene la conversión cerca del tipo "sucio" de origen (DTO/Entity), sin que el modelo de dominio (`Player`) necesite saber nada sobre de dónde vino el dato — la capa `domain` sigue siendo Kotlin puro, sin imports de Ktor ni SQLDelight, y la extension function es lo que traduce entre ambos mundos sin romper esa separación.