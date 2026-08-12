# Generics y Varianza (in / out)

## 1. Qué es

Generics y varianza (`in`/`out`) controlan si un tipo genérico parametrizado es compatible en relaciones de subtipo.

- **`out T` (covarianza):** el tipo solo se produce (se devuelve), nunca se consume como parámetro.
- **`in T` (contravarianza):** el tipo solo se consume (se recibe como parámetro), nunca se devuelve.

```kotlin
interface ReadOnlyRepository<out T> {
    fun getAll(): List<T>
}
```

## 2. El problema que resuelve

Sin varianza declarada, los tipos genéricos son invariantes por defecto: `Repository<Player>` y `Repository<Any>` no tienen ninguna relación de subtipo entre sí, aunque `Player` sea subtipo de `Any`. Esto es innecesariamente rígido para interfaces de solo lectura, donde en la práctica sí debería poder tratarse un `Repository<Player>` como un `Repository<Any>` sin ningún riesgo real de romper type safety, porque nunca se inserta nada en él.

## 3. Ejemplo mínimo comentado

```kotlin
// covarianza: Repository solo PRODUCE T, nunca lo recibe como parámetro
interface ReadOnlyRepository<out T> {
    fun getAll(): List<T>
}

val repo: ReadOnlyRepository<Player> = PlayerRepositoryImpl()
val anyRepo: ReadOnlyRepository<Any> = repo  // válido gracias a "out"
```

```kotlin
// contravarianza: un comparador que consume Animal sirve también para Dog
interface Comparator<in T> {
    fun compare(a: T, b: T): Int
}

class AnimalComparator : Comparator<Animal> { /* ... */ }
val dogComparator: Comparator<Dog> = AnimalComparator()  // válido gracias a "in"
```

## 4. Matriz de criterio

**Usar `out T` cuando:** tu interfaz/clase genérica solo expone métodos que **devuelven** `T` (nunca lo reciben como parámetro) — ganás flexibilidad de subtipos sin ningún riesgo, porque el compilador garantiza en compile-time que no hay forma de insertar el tipo incorrecto.

**Usar `in T` cuando:** tu interfaz/clase genérica solo **consume** `T` como parámetro (nunca lo devuelve) — típico en comparadores, callbacks, o consumidores de eventos.

**NO marcar varianza cuando:** el tipo genérico se usa tanto para leer como para escribir (`MutableList<T>` es el ejemplo canónico) — si fuera covariante, podrías insertar un tipo incorrecto en lo que el código cree que es una colección de otro tipo, rompiendo type safety en runtime. Por eso `MutableList<T>` es invariante.

**Trade-off real:** declarar `out`/`in` es una decisión de diseño de API que compromete el contrato hacia adelante — si marcás `out T` y más adelante necesitás agregar un método que reciba `T` como parámetro, dejás de poder compilar (el compilador te lo va a impedir), así que hay que pensar la forma final de la interfaz antes de decidir la varianza.

## 5. Caso trampa

Asumir que se puede marcar `out` en cualquier interfaz "de solo lectura aparente" sin revisar todos sus métodos:

```kotlin
interface Repository<out T> {
    fun getAll(): List<T>
    fun save(item: T) // ❌ ERROR de compilación: 'T' está en posición 'in' pero el parámetro es 'out'
}
```

La trampa es pensar "esto es un Repository, es de lectura" sin notar que `save(item: T)` consume `T` como parámetro — el compilador rechaza esto directamente, y es la pregunta típica de entrevista: "¿por qué `Repository<out T>` tiene sentido pero `Repository<T>` con un método que recibe `T` no?" → porque covarianza solo es segura si `T` nunca aparece en posición de parámetro de entrada.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, las interfaces de repository del `domain` que son estrictamente de lectura (por ejemplo, un futuro `ReadOnlyStatsRepository` que solo expone `getAll(): Flow<List<Stat>>`) podrían declararse con `out T` para ganar flexibilidad de subtipos sin costo. En la práctica, la mayoría de los repositories de Timbax (`PlayerRepository`) también tienen métodos de escritura (`save`, `update`), por lo que no son candidatos a `out T` — son invariantes por necesidad, ya que consumen y producen `T` a la vez.