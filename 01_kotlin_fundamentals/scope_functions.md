# Scope Functions (let, run, apply, also, with)

## 1. Qué es

Las scope functions (`let`, `run`, `apply`, `also`, `with`) son funciones de la stdlib de Kotlin que ejecutan un bloque de código en el contexto de un objeto, sin necesidad de nombrarlo repetidamente. Se diferencian en dos ejes: cómo referencian al objeto (`it` vs `this`) y qué devuelven (el objeto mismo vs el resultado del lambda).

| Función | Referencia | Devuelve |
|---|---|---|
| `let` | `it` | resultado del lambda |
| `run` | `this` | resultado del lambda |
| `apply` | `this` | el objeto mismo |
| `also` | `it` | el objeto mismo |
| `with` | `this` | resultado del lambda |

## 2. El problema que resuelve

Sin scope functions, operar sobre un objeto recién creado (configurarlo, transformarlo, o actuar solo si no es null) obliga a repetir el nombre de la variable en cada línea, o a declarar variables intermedias solo para un uso puntual. Esto ensucia el código con ruido que no aporta información — el lector tiene que rastrear el mismo identificador línea por línea en vez de ver de un vistazo "todo esto es sobre el mismo objeto".

## 3. Ejemplo mínimo comentado

```kotlin
val player = Player(id = "1", name = "", score = 0).apply {
    // "this" es el player recién creado — útil para builder-style
}
```

```kotlin
val nameLength = player.name.let {
    // "it" es player.name — devuelve un resultado distinto (Int)
    if (it.isEmpty()) 0 else it.length
}
```

```kotlin
// nullable?.let — patrón típico para actuar solo si no es null
selectedPlayer?.let { p ->
    println("Jugador seleccionado: ${p.name}")
}
```

## 4. Matriz de criterio

**Usar `apply` cuando:** estás configurando propiedades de un objeto recién creado y querés devolver ese mismo objeto (builder-style, inicialización de un `Modifier` o similar).

**Usar `also` cuando:** necesitás un side-effect (loguear, validar) sin alterar el flujo principal, y preferís `it` porque vas a pasar el objeto a otra función o referenciarlo explícitamente.

**Usar `let` cuando:** transformás un valor en otro tipo de resultado, o necesitás ejecutar código solo si un valor nullable no es null (`nullable?.let { }`).

**Usar `run` cuando:** agrupás inicialización + cálculo que termina en un valor de retorno, sin necesidad de un receptor explícito fuera del bloque.

**Usar `with` cuando:** vas a llamar varias funciones sobre el mismo objeto no-nullable y no necesitás el estilo fluido de un receptor implícito en cadena (no es extension function, se llama `with(objeto) { }`).

**NO usar scope functions cuando:** anidás varias al punto de que ya no queda claro a qué objeto refiere `it` o `this` en cada nivel — ahí es más legible nombrar variables explícitas.

**Trade-off real:** ganás concisión y legibilidad de intención, pero perdés en debuggeabilidad si anidás demasiado (el debugger muestra lambdas, no nombres de variable claros) y en legibilidad si quien lee el código no tiene el hábito internalizado de estas 5 funciones.

## 5. Caso trampa

Confundir `apply`/`also` (devuelven el objeto) con `let`/`run` (devuelven el resultado del lambda) es el error más común. Ejemplo trampa:

```kotlin
val result = player.apply {
    "esto no se devuelve"
}
// result es el Player original, NO el String — apply siempre devuelve el objeto
```

Si la intención era obtener el String, la función correcta era `let` o `run`, no `apply`.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, un patrón típico en mappers de la capa `data` es usar `also` para loguear una transformación sin cortar la cadena:

```kotlin
fun PlayerDto.toDomain(): Player = Player(id, name, score)
    .also { Logger.d("Mapeado DTO -> Domain: ${it.id}") }
```

Y `apply` aparece naturalmente al construir un `MutableStateFlow` inicial o configurar un `HttpClient` de Ktor en el módulo de DI, donde varias propiedades se setean sobre el mismo objeto recién creado antes de exponerlo.