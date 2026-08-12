# Keywords Reservadas de Kotlin

## 1. Qué es

Kotlin clasifica sus palabras reservadas en 4 categorías según cuán "estrictamente" ocupan el nombre: **Hard Keywords** (siempre son palabra clave, nunca se pueden usar como identificador salvo con backticks), **Soft Keywords** (son palabra clave solo en su contexto específico, en cualquier otro lado son identificadores normales), **Modifier Keywords** (actúan como palabra clave dentro de una lista de modificadores de una declaración) y **Special Identifiers** (definidos por el compilador solo en contextos muy puntuales, como `it` o `field`).

## 2. El problema que resuelve

Sin esta categorización, no hay forma de predecir si un nombre como `get`, `value`, `by` o `data` está disponible como nombre de variable/función propio o si va a chocar con el compilador. Entender la categoría de cada keyword evita dos errores típicos: usar por accidente un nombre reservado como identificador y no entender por qué falla, o asumir que una palabra es "reservada siempre" cuando en realidad es soft/modifier y solo aplica en un contexto puntual.

## 3. Ejemplo mínimo comentado

```kotlin
// "data" es Modifier Keyword — reservado SOLO antes de "class"
data class Player(val id: String)

// pero "data" fuera de ese contexto es un identificador válido
val data = "esto compila perfecto"
```

```kotlin
// "by" es Soft Keyword — reservado SOLO en delegación
var expanded by remember { mutableStateOf(false) }

// "by" como identificador en cualquier otro contexto también sería válido
// (aunque en la práctica casi nadie lo usa así, por claridad)
```

## 4. Matriz de criterio

**Usar backticks (`` `nombre` ``) cuando:** necesitás usar una Hard Keyword como identificador de forma forzada — típicamente en interop con Java/JS donde el nombre ya viene dado y no se puede cambiar (ej. `` `class` ``, `` `object` ``). Es un recurso de último caso, no una práctica recomendada para nombres propios.

**Consultar la categoría antes de nombrar algo cuando:** el nombre elegido coincide con una palabra de la guía completa (`Keywords_guide`) y no compila como esperás — la causa casi siempre es que es Hard Keyword (no se puede usar tal cual) vs Soft/Modifier (sí se puede, salvo en su contexto específico).

**NO memorizar las ~60 keywords de memoria:** el valor real no es recordar la lista completa, sino tener el criterio de "si algo no compila como nombre de variable, probablemente es una keyword reservada — chequeo la categoría" y saber dónde está la referencia completa (`Keywords_guide`) para consultar rápido.

**Trade-off real:** ninguno de diseño — esto es una convención fija del lenguaje. El único trade-off real es de tiempo: memorizar todo de antemano no rinde tanto como tener el criterio de categorías y una referencia a mano para el caso puntual que surja.

## 5. Caso trampa

Asumir que porque una palabra "suena reservada" (aparece coloreada en el IDE) es Hard Keyword y por lo tanto nunca se puede usar como nombre:

```kotlin
// "get" es Soft Keyword — reservado solo como declarador de getter
val Player.displayName: String
    get() = if (name.isBlank()) "Anónimo" else name

// pero esto también compila perfecto, "get" como identificador normal:
val get = "esto es válido, get es soft keyword"
```

La trampa inversa también existe: asumir que porque algo compiló como identificador en un contexto, va a compilar en cualquier lado — `field` es Special Identifier, solo tiene su significado especial (acceso al backing field) dentro de un `get()`/`set()` personalizado; fuera de ahí es un identificador cualquiera sin ningún significado mágico.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, la categoría más relevante en el día a día es **Modifier Keywords**: `data`, `sealed`, `private`, `internal`, `suspend`, `operator` aparecen constantemente en el modelo de dominio, los Contracts de MVI y los UseCases (`operator fun invoke()`). Entender que son modifiers (y no funciones o palabras sueltas) explica por qué siempre aparecen pegadas a una declaración específica (`class`, `fun`, `val`) y nunca solas. La referencia completa de las ~60 keywords con ejemplos está en `Keywords_guide`, documento fuente de este repo.