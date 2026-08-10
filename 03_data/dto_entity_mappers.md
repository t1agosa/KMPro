# DTO / Entity / Mappers

## 1. Qué es

DTO (Data Transfer Object) y Entity son los modelos "sucios" de la capa `data` — atados a una tecnología concreta y anotados con lo que esa tecnología necesita para funcionar. El **DTO** representa la forma en que un dato viaja por la red (`@Serializable` de Kotlinx Serialization, hablando con Ktor). La **Entity** representa la forma en que un dato se guarda en disco (`@Entity` de Room, o la clase generada por SQLDelight a partir de un `.sq`). Ninguno de los dos cruza jamás hacia `domain` ni hacia `presentation`/`UI` — el puente entre estos modelos "sucios" y el modelo de dominio puro (`Player`) lo hacen los **Mappers**: funciones de extensión que traducen de un lado al otro.

```kotlin
// data — DTO (viene de la red)
@Serializable
data class PlayerDto(
    val id: String,
    val name: String,
    val score: Int
)

// data — Entity (vive en SQLDelight/Room)
data class PlayerEntity(
    val id: String,
    val name: String,
    val score: Long // SQLDelight guarda enteros como Long
)

// domain — Model puro (ya documentado en model.md)
data class Player(
    val id: String,
    val name: String,
    val score: Int
)

// data — Mapper
fun PlayerDto.toDomain(): Player = Player(id = id, name = name, score = score)
fun PlayerEntity.toDomain(): Player = Player(id = id, name = name, score = score.toInt())
fun Player.toEntity(): PlayerEntity = PlayerEntity(id = id, name = name, score = score.toLong())
```

## 2. El problema que resuelve

Si `Player` (el modelo de dominio) tuviera directamente las anotaciones `@Serializable` o `@Entity`, domain dejaría de ser "puro Kotlin sin dependencias externas" — pasaría a depender de Kotlinx Serialization o de Room/SQLDelight, dos tecnologías que domain no debería ni conocer. Peor todavía: un cambio en la forma en que la API entrega el JSON (por ejemplo, que agregue un campo `avatar_url` que a domain no le importa) terminaría forzando cambios en cascada sobre `UseCases`, `ViewModels` y hasta la UI, cuando en realidad ese cambio debería quedar totalmente contenido dentro de la capa `data`. DTO/Entity + Mappers resuelven esto separando "cómo llega/se guarda el dato" de "qué es el dato para el negocio" — cada capa habla su propio idioma, y los mappers son los únicos que entienden los dos.

## 3. Ejemplo mínimo comentado

```kotlin
// DTO: forma exacta del JSON que devuelve la API. Puede tener campos que domain ni necesita.
@Serializable
data class PlayerDto(
    val id: String,
    val name: String,
    val score: Int,
    @SerialName("avatar_url") val avatarUrl: String? = null // domain no lo usa, se descarta acá
)

// Entity: forma exacta de la tabla en SQLDelight/Room.
data class PlayerEntity(
    val id: String,
    val name: String,
    val score: Long
)

// Mapper DTO -> Domain: se descarta lo que domain no necesita (avatarUrl)
fun PlayerDto.toDomain(): Player = Player(
    id = id,
    name = name,
    score = score
)

// Mapper Domain -> Entity: se adapta el tipo (Int -> Long) que pide SQLDelight
fun Player.toEntity(): PlayerEntity = PlayerEntity(
    id = id,
    name = name,
    score = score.toLong()
)

// Mapper Entity -> Domain: camino inverso, para cuando se lee de la DB
fun PlayerEntity.toDomain(): Player = Player(
    id = id,
    name = name,
    score = score.toInt()
)
```

El patrón se repite siempre igual: `algo.toX()`, donde `X` es la capa destino. Nunca al revés (nunca `Player.fromDto(dto)` como función estática) — como extension function, el mapper queda pegado a quien lo origina, no a quien lo recibe.

## 4. Matriz de criterio

**Mapper como extension function (`Dto.toDomain()`):**
- Usar cuando: es el caso general — mappers simples, 1 a 1, sin dependencias externas para hacer la conversión.
- NO usar cuando: el mapeo necesita contexto externo para resolverse (ej: convertir un `score` relativo a absoluto necesitando el `GameConfig` actual) — ahí conviene una función normal que reciba ese contexto como parámetro, en vez de forzarlo en una extension function.
- Trade-off: extension functions son livianas y muy legibles en el punto de uso (`dto.toDomain()`), pero si el mapeo se vuelve complejo (múltiples fuentes, lógica condicional pesada) mezclarlo todo en una sola función extension puede volverse difícil de testear en aislamiento — ahí vale la pena promoverlo a una clase `Mapper` con su propio test.

**Un DTO/Entity por tecnología, sin reusar el mismo modelo para las dos:**
- Usar cuando: siempre — aunque `PlayerDto` y `PlayerEntity` terminen teniendo exactamente los mismos campos hoy, son conceptualmente cosas distintas (uno describe un contrato de red, el otro un esquema de tabla) y evolucionan por razones distintas.
- NO usar cuando: prácticamente nunca conviene "ahorrarse" esta separación — es una trampa común de proyectos chicos que después cuesta desarmar cuando el DTO y la Entity empiezan a divergir de verdad.
- Trade-off: parece código duplicado al principio (dos clases casi idénticas), pero esa "duplicación" es la que evita que un cambio en el JSON de la API rompa el schema de la DB, o viceversa.

**Descartar campos en el mapper vs. traerlos igual a domain "por las dudas":**
- Usar cuando: domain solo debería tener las propiedades que el negocio realmente usa — si `avatarUrl` no se usa en ninguna regla de negocio ni se muestra en UI todavía, no entra a `Player`.
- NO usar cuando: si un campo hoy no se usa pero se sabe con certeza que se va a necesitar pronto (no "por las dudas" especulativo), ahí sí tiene sentido agregarlo a domain de una.
- Trade-off: ser estricto acá mantiene `Player` limpio y con propósito claro, pero implica tocar el mapper (y potencialmente el modelo de domain) cada vez que una nueva necesidad de negocio aparece — es el costo aceptado de mantener domain puro.

## 5. Caso trampa

Un mapper que parece inofensivo pero esconde pérdida silenciosa de datos:

```kotlin
// DTO real que devuelve la API — score puede venir null si el jugador nunca jugó
@Serializable
data class PlayerDto(
    val id: String,
    val name: String,
    val score: Int? // nullable en el contrato real de la API
)

// ❌ trampa: el mapper "resuelve" el null con un default silencioso
fun PlayerDto.toDomain(): Player = Player(
    id = id,
    name = name,
    score = score ?: 0 // ¿es 0 lo mismo que "nunca jugó"? el mapper decidió que sí, sin que nadie lo pida
)
```

El código compila, no lanza excepciones, y en la mayoría de los tests "funciona". El problema es semántico: el mapper tomó una decisión de negocio (tratar "sin score" como "score cero") que en realidad le correspondía a domain o a una regla explícita, no a una conversión de tipos. Si más adelante alguien agrega una feature de ranking, un jugador que "nunca jugó" (score null) va a aparecer empatado en el último lugar con uno que efectivamente sacó 0 puntos jugando — son casos distintos que el mapper fusionó sin que quede rastro de la decisión en ningún lado.

La regla: si un mapper necesita "inventar" un valor por default para resolver un null, esa decisión tiene que ser explícita y visible — o modelarse en domain (`score: Int?` también ahí, si "sin jugar" es un estado real del negocio), o resolverse en un UseCase donde la regla quede documentada, no escondida dentro de una extension function de una línea.

## 6. Conexión

En Timbax, cada fuente de datos (`PlayerApi` para lo que eventualmente sincroniza con backend, `PlayerDao`/SQLDelight para lo local) tiene su propio DTO/Entity, y `PlayerRepositoryImpl` (visto en `repository_impl.md`) es quien invoca estos mappers en cada punto de entrada/salida — nunca expone un DTO o Entity fuera de sus propios métodos. Esto conecta directo con la Dependency Rule del machete: los mappers son el mecanismo concreto que hace cumplir "nadie fuera de `data` debería saber si un dato vino de una API o de una tabla SQL" — el `Player` que llega a un `UseCase` es siempre el mismo tipo, sin importar cuál mapper lo produjo.