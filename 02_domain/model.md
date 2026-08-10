# model.md

## 1. Qué es

El **Model** es la representación pura de un concepto del negocio, escrita como `data class` de Kotlin sin ninguna anotación ni dependencia de librería externa (nada de `@Entity`, `@Serializable`, `@Composable`). Vive en `domain/model/` y es el tipo que circula por Presentation y por los UseCases — es el "idioma" común de toda la app.

```kotlin
data class Player(
    val id: String,
    val name: String,
    val score: Int
)
```

## 2. El problema que resuelve

Sin un Model de dominio separado, la tentación natural es usar directamente el DTO de la API o la Entity de la base de datos en toda la app — incluida la UI. El problema aparece apenas cambia algo en la fuente de datos: si el backend renombra un campo, o migrás de Room a SQLDelight, esa anotación técnica (`@SerialName`, `@Entity`) se filtra hasta el Composable, y un cambio de infraestructura termina rompiendo pantallas que no tienen nada que ver con la razón del cambio.

El Model de dominio actúa como una frontera: todo lo que está "afuera" (red, DB) puede cambiar de forma completamente libre, porque nunca cruza esa frontera sin pasar antes por un Mapper.

## 3. Ejemplo mínimo comentado

```kotlin
// domain/model/Player.kt
// Puro Kotlin: sin @Entity (Room/SQLDelight), sin @Serializable (Ktor), sin nada de UI.
data class Player(
    val id: String,
    val name: String,
    val score: Int
)
```

Comparado con lo que NO debería pasar a Presentation:

```kotlin
// data/remote/dto/PlayerDto.kt
// Esto es un modelo "sucio" — sabe que existe una API. Nunca sale de la capa data.
@Serializable
data class PlayerDto(
    @SerialName("player_id") val id: String,
    @SerialName("player_name") val name: String,
    val score: Int
)
```

## 4. Matriz de criterio

**Usar Model de dominio cuando:**
- El dato representa un concepto real del negocio que UseCases, ViewModels o UI necesitan manipular (`Player`, `Round`, `GameSession`).
- Más de una fuente de datos (remote + local) puede terminar produciendo el mismo concepto — el Model es el punto de convergencia único.

**NO crear un Model de dominio separado cuando:**
- El dato es puramente técnico de una capa (ej: un `PaginationCursor` interno de un repositorio remoto) y jamás se expone hacia arriba — ahí no hace falta modelarlo en domain, viviría directamente en data.
- Es un tipo tan trivial y de un solo uso que envolverlo en un modelo propio agrega ceremonia sin ganancia real (ej: un `String` de configuración que no tiene comportamiento ni validación asociada).

**Trade-off real:** cada Model de dominio implica un Mapper por cada fuente de datos que lo produzca. Con una sola fuente de datos, el mapeo puede sentirse redundante a corto plazo — pero es el costo de tener la frontera lista para cuando aparezca una segunda fuente (cache local, otro backend), que en apps reales casi siempre termina apareciendo.

## 5. Caso trampa

Timbax guarda las partidas en SQLDelight. La entidad generada por SQLDelight se llama `PlayerEntity` y tiene exactamente los mismos campos que `Player` de domain: `id`, `name`, `score`. La tentación es "ahorrarse" el Mapper y usar `PlayerEntity` directo en el ViewModel, ya que "son iguales".

La respuesta correcta es mapear igual, aunque hoy sean idénticos. La razón: la igualdad actual es coincidencia, no garantía. El día que SQLDelight cambie el tipo generado (por ejemplo, `score` pasa a ser `Long` porque la query hace un `SUM()`), o el día que agregues un campo técnico a la tabla (`created_at`, `synced` para offline-first) que domain no necesita, el `PlayerEntity` deja de ser igual a `Player` — y si no había Mapper, ese cambio de infraestructura se propaga directo hasta la UI sin que el compilador te avise dónde hay que arreglar algo.

## 6. Conexión

En Timbax, `Player` es el Model que atraviesa toda la cadena: `PlayerDao` (SQLDelight) → `PlayerRepositoryImpl` → mapper `PlayerEntity.toDomain()` → `Player` limpio que llega al UseCase y de ahí al `PlayersState` que consume la UI.



