# modifiers_fundamentos.md

## 1. Qué es

`Modifier` es un objeto inmutable que decora o transforma un composable: le agrega comportamiento (padding, tamaño, click, scroll, fondo) sin que el composable necesite exponer un parámetro propio para cada una de esas cosas. Se construye encadenando funciones de extensión sobre `Modifier` (`Modifier.padding(8.dp).fillMaxWidth()`), y esa cadena se pasa como parámetro `modifier` al composable que la va a aplicar.

Cada función de la cadena envuelve a la anterior, formando una secuencia ordenada de transformaciones — no una lista de propiedades sin orden, como sería un `style` en CSS o un `Bundle` de atributos.

## 2. El problema que resuelve

Sin `Modifier`, cada composable (`Text`, `Button`, `Card`) necesitaría exponer decenas de parámetros propios para cubrir cualquier combinación posible de padding, tamaño, click, borde, fondo, etc. — una API inmanejable, y encima cada composable tendría que reimplementar esa lógica por separado.

`Modifier` desacopla "qué es el composable" de "cómo se comporta/luce en este lugar puntual". `Text` solo sabe dibujar texto; el padding, el ancho, o si es clickeable, se lo decide quien lo usa, vía `Modifier` — no el propio `Text`. Esto es, en esencia, el mismo principio de **state hoisting** aplicado al styling: elevar la decisión hacia quien llama, no hardcodearla adentro.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayerCard(
    player: Player,
    onClick: (String) -> Unit,
    modifier: Modifier = Modifier // convención: siempre con default vacío
) {
    Card(
        modifier = modifier
            .fillMaxWidth()              // 1. ocupa todo el ancho disponible
            .padding(horizontal = 16.dp) // 2. margen externo, respecto al padre
            .clickable { onClick(player.id) } // 3. hace clickeable TODO lo anterior
            .padding(12.dp)              // 4. padding interno, respecto al contenido
    ) {
        Text(player.name)
    }
}
```

El orden importa: el `padding(horizontal = 16.dp)` del paso 2 se aplica *antes* de que la card sea clickeable, así que ese margen queda fuera del área clickeable. El `padding(12.dp)` del paso 4 se aplica *después* de `clickable`, así que ese espacio interno sí es parte del área que responde al click. Invertir el orden de estos modifiers cambia el comportamiento real, no solo el resultado visual.

## 4. Matriz de criterio

**Orden de la cadena**
- Usar cuando: cada modifier se lee de afuera hacia adentro — el primero afecta cómo el composable se posiciona respecto a su padre, los últimos afectan su contenido interno.
- NO asumir que el orden es "solo estético" — `padding` antes o después de `clickable`/`background`/`border` cambia qué área queda afectada.
- Trade-off: ganás control preciso, pero perdés margen de error — un `Modifier` mal ordenado no tira warning, produce un bug visual/funcional silencioso.

**Modifiers scope-specific (`weight`, `align` dentro de `Row`/`Column`, `matchParentSize` dentro de `Box`)**
- Usar cuando: necesitás que un hijo se comporte en relación al layout padre (repartir espacio proporcionalmente con `weight`, alinearse dentro de un `Box` con `align`).
- NO usar cuando: intentás usar `Modifier.weight()` fuera de un `RowScope`/`ColumnScope` — no compila, porque `weight` es una extension function que solo existe en ese scope específico.
- Trade-off: acoplan el composable hijo a saber en qué tipo de padre está — un composable reusable no debería asumir de antemano que un modifier scope-specific va a estar disponible; eso lo decide quien lo llama.

**`Modifier` como parámetro con default (`modifier: Modifier = Modifier`)**
- Usar cuando: siempre, en todo composable reusable — es la convención oficial de Compose.
- NO usar cuando: nunca deberías omitirlo en un composable propio, salvo que sea genuinamente una hoja terminal sin necesidad de personalización externa (raro).
- Trade-off: ninguno real — es pura disciplina de API design; omitirlo rompe la componibilidad del composable con el resto del árbol.

## 5. Caso trampa

```kotlin
// Intención: un box clickeable de 100dp con padding interno de 8dp
Box(
    modifier = Modifier
        .padding(8.dp)
        .size(100.dp)
        .clickable { onClick() }
        .background(Color.Red)
)
```

La trampa: se podría asumir que el área roja y clickeable mide 100dp, con 8dp de aire alrededor. En realidad, `padding(8.dp)` se aplica primero, *reduciendo* el espacio disponible para el `size(100.dp)` que viene después si el padre tiene un tamaño acotado — o, si el padre no lo limita, el resultado final tiene 8dp de margen invisible *más* 100dp de tamaño, dando un total de 116dp de ancho ocupado, pero el `background` y el `clickable` solo cubren los 100dp internos, no el padding. El bug típico: el usuario "casi" toca la card y no pasa nada, porque tocó la zona de `padding`, que quedó *fuera* del `clickable`. La solución no es memorizar este caso puntual, sino siempre preguntarse, modifier por modifier: "¿qué área abarca todo lo que viene antes de este punto en la cadena?"

## 6. Conexión con arquitectura real

En Timbax, la convención `modifier: Modifier = Modifier` en cada composable (`PlayerCard`, `GameScoreText`, etc.) es lo que permite que `PlayersScreen` decida el padding y el ancho de cada card desde afuera, sin que `PlayerCard` necesite saber en qué pantalla va a vivir. Es la misma idea de **state hoisting** que ya vimos en `layouts_basicos.md`, pero aplicada al styling en vez de al estado: la responsabilidad de "cómo se ve en este contexto" se eleva hacia el composable padre, dejando al hijo reusable y desacoplado del lugar donde se use.