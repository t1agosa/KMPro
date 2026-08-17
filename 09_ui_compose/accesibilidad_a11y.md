# Accesibilidad (a11y)

## 1. Qué es

Accesibilidad (a11y) es el conjunto de prácticas y APIs que permiten que una app sea usable por personas con discapacidades visuales, motoras o cognitivas, típicamente a través de servicios de asistencia como TalkBack (el lector de pantalla de Android) o Switch Access (navegación sin tocar la pantalla, vía switches físicos o gestos de barrido). En Compose, la pieza central es el **árbol de semántica** (semantics tree): un árbol paralelo al árbol de UI visual que describe el "significado" de cada elemento (es un botón, dice "Favorito", está seleccionado) para que los servicios de accesibilidad puedan interpretarlo sin necesidad de "ver" píxeles.

## 2. El problema que resuelve

Sin un árbol de semántica explícito, un lector de pantalla no tiene forma de saber qué es cada elemento visual: un ícono es solo un conjunto de píxeles, un `Row` con texto e ícono son elementos sueltos sin relación aparente entre sí. Compose construye automáticamente semántica razonable para los componentes estándar (un `Button` ya se anuncia como botón, un `Text` lee su contenido), pero en cuanto se arma algo custom (una card compuesta por ícono + texto + badge, un gráfico, un gesto propio), esa semántica automática no alcanza y hay que describirla a mano — si no se hace, la app queda invisible o confusa para cualquiera que dependa de TalkBack/Switch Access, lo cual además de ser una barrera real es, en mercados como la UE, un requisito legal (European Accessibility Act, vigente desde 2025).

## 3. Ejemplo mínimo comentado

```kotlin
// Ícono decorativo dentro de un botón: contentDescription = null porque el texto
// del botón ya comunica la acción — el ícono no aporta información nueva
Button(onClick = { onFavoriteClick(player.id) }) {
    Icon(Icons.Filled.Star, contentDescription = null)
    Text("Marcar favorito")
}

// Ícono que ES la única fuente de información (no hay texto al lado):
// acá contentDescription es obligatorio, describe la ACCIÓN, no el ícono en sí
IconButton(onClick = { onDeletePlayer(player.id) }) {
    Icon(Icons.Filled.Delete, contentDescription = "Eliminar a ${player.name}")
}
```

```kotlin
// Componente custom: agrupa varios elementos hijos en un solo nodo semántico,
// para que TalkBack lo anuncie como una sola unidad y no como piezas sueltas
@Composable
fun PlayerScoreCard(player: Player, score: Int) {
    Row(
        modifier = Modifier.semantics(mergeDescendants = true) {}
    ) {
        Avatar(player.avatarUrl)
        Column {
            Text(player.name)
            Text("Puntaje: $score")
        }
    }
}
```

`contentDescription = null` en el primer caso no es un olvido — es una decisión explícita: le dice a Compose "este elemento es decorativo, no lo anuncies por separado", evitando que TalkBack lea dos veces la misma información ("Estrella. Marcar favorito." en vez de solo "Marcar favorito.").

## 4. Matriz de criterio

**`contentDescription = null` vs. una descripción real:**
- Usar `null` cuando: el ícono es puramente decorativo o ya está acompañado de un texto que comunica lo mismo (como el botón "Marcar favorito") — evita redundancia al lector de pantalla.
- Usar una descripción real cuando: el elemento es la única fuente de información de esa acción (un `IconButton` sin texto visible) — y la descripción debe describir la **acción** ("Eliminar a Juan"), no el ícono en sí ("ícono de tacho de basura").
- Trade-off: ninguno real — es la regla correcta en cada caso; el error común es dejar el default o poner una descripción genérica ("botón") que no aporta nada.

**`Modifier.semantics(mergeDescendants = true)` vs. dejar que cada hijo se anuncie por separado:**
- Usar cuando: varios elementos hijos (ícono + texto + badge) forman conceptualmente una sola unidad de información — como `PlayerScoreCard`, que debería anunciarse como una sola unidad y no como anuncios sueltos.
- Usar con cuidado cuando: alguno de los hijos es interactuable pero fue armado con un gesto custom (`Modifier.pointerInput { detectTapGestures { } }`) en vez de `clickable`/`toggleable`. Los modifiers estándar declaran su propia semántica y quedan automáticamente protegidos de ser absorbidos por el merge del padre — pero un gesto custom de bajo nivel, sin semántica propia, sí puede quedar silenciosamente tragado dentro del nodo mergeado (ver Caso trampa).
- Trade-off: mergear de más no rompe nada mientras los hijos interactivos usen las APIs estándar de Compose (que ya se auto-protegen); el riesgo real aparece con gestos custom de bajo nivel.

**Tamaño mínimo de touch target (48dp) — dejarlo por default vs. ajustarlo manualmente:**
- Dejarlo por default cuando: se usan componentes de Material (`IconButton`, `Checkbox`, `Switch`, etc.) — Compose ya aplica automáticamente una región de touch invisible de al menos 48dp alrededor del elemento, aunque el ícono visible sea más chico (ej: un ícono de 24dp sigue teniendo un área táctil real de 48dp).
- Ajustarlo manualmente cuando: se arma un componente completamente custom con `Modifier.clickable` sobre algo visualmente más chico que 48dp, o cuando varios elementos chicos quedan muy juntos entre sí — ahí el área táctil invisible de cada uno puede superponerse con la del vecino, y hay que espaciarlos a mano.
- Trade-off: cumplir el mínimo no tiene costo real de diseño (es un área invisible, no cambia el look visual), pero ignorarlo es causa común de bugs de usabilidad — no solo para quien tiene una discapacidad motora, para cualquiera con el celular en una mano y el pulgar grande.

**`clearAndSetSemantics {}` vs. `semantics {}` (agregar) vs. no tocar nada:**
- Usar `semantics {}` (sin `clearAndSet`) cuando: querés **agregar** información semántica sin pisar lo que Compose ya generó automáticamente para ese componente — es el caso general.
- Usar `clearAndSetSemantics {}` cuando: necesitás **reemplazar por completo** la semántica de un nodo y sus descendientes, típicamente cuando la lectura automática expondría detalles de implementación irrelevantes para el usuario.
- Trade-off: `clearAndSetSemantics` es poderosa pero peligrosa — borra toda la semántica previa (incluida la de los hijos), así que usarla de más deja huecos de información reales. Se recomienda con moderación, solo cuando `semantics {}` no alcanza.

## 5. Caso trampa

Un ícono interactivo armado con un gesto custom (`pointerInput` + `detectTapGestures`) en vez de `clickable`, dentro de una fila con `mergeDescendants = true`:

```kotlin
// ❌ trampa: el ícono de "más info" usa un gesto custom, no clickable()
@Composable
fun PlayerRow(player: Player, onExpandInfo: () -> Unit) {
    Row(
        modifier = Modifier.semantics(mergeDescendants = true) {}
    ) {
        Text(player.name)
        Icon(
            Icons.Filled.Info,
            contentDescription = "Ver más info",
            modifier = Modifier.pointerInput(Unit) {
                detectTapGestures { onExpandInfo() } // gesto custom, sin clickable()
            }
        )
    }
}
```

Para un usuario sin discapacidad, esto funciona perfecto: toca el ícono, dispara `onExpandInfo()`, todo bien. El problema aparece con TalkBack activado. `clickable`/`toggleable` no son solo manejo de gestos — internamente también registran una acción de accesibilidad (`onClick`) en el árbol de semántica, que es lo que le permite a TalkBack ofrecer "doble tap para activar" sobre ese elemento. `pointerInput` + `detectTapGestures` es pura detección de gesto de bajo nivel: no registra ninguna acción semántica. El resultado es que la fila entera se mergea y TalkBack anuncia "Juan. Ver más info." — el `contentDescription` del ícono queda absorbido como texto informativo dentro del nodo mergeado, pero no existe ninguna acción activable adjunta a esa parte. El usuario escucha que "hay más info" pero no tiene forma de pedirla: la funcionalidad existe visualmente y funciona al tacto, pero es invisible y no-operable para quien navega con TalkBack.

La señal de alarma: cualquier `Modifier.pointerInput`/`detectTapGestures` usado para algo interactuable, sin acompañarlo de `Modifier.semantics { onClick { } }` (o directamente reemplazado por `clickable()`), es candidato a este problema — pasa cualquier QA manual con el dedo sin levantar sospechas, y solo aparece al probar la pantalla con TalkBack activado.

## 6. Conexión con Timbax

Timbax usa componentes de Material3 (botones, cards, checkboxes) para casi toda su UI, lo cual da gran parte de la semántica base gratis — los componentes estándar ya declaran su propia semántica y su touch target mínimo de 48dp sin que el desarrollador tenga que pensar en eso. Donde sí hace falta prestar atención a mano es en los componentes custom del juego — la card de cada jugador con su puntaje (`PlayerScoreCard`, igual al ejemplo de este archivo) y cualquier gesto custom que se agregue a futuro (por ejemplo, un swipe para eliminar un jugador de una partida) necesita, además del gesto visual, su semántica explícita para no volverse invisible para quien juega con TalkBack activado. Esto conecta directo con `listas_lazy.md` (cada ítem de una `LazyColumn` de jugadores es candidato a mergear su semántica correctamente) y con `material3_componentes_comunes.md` (la base de por qué apoyarse en componentes estándar resuelve la mayor parte de la accesibilidad sin código adicional).