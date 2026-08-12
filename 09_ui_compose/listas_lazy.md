
# listas_lazy.md

## 1. Qué es

`LazyColumn` y `LazyRow` son las versiones "perezosas" (lazy) de `Column` y `Row`: en vez de componer todos sus hijos de una, solo componen y dibujan los ítems que efectivamente están (o van a estar pronto) visibles en pantalla, reciclando el trabajo a medida que el usuario scrollea. Se construyen con un DSL propio (`LazyListScope`) en vez de recibir composables hijos directos: `items()`, `item()`, `itemsIndexed()`.

## 2. El problema que resuelve

`Column` compone TODOS sus hijos apenas se le pide, sin importar cuántos sean ni si están visibles. Con 20 jugadores no pasa nada; con una lista de 5.000 partidas históricas, eso significa crear 5.000 composables en memoria de una sola vez — la mayoría invisibles — con el consiguiente costo de composición, medición y memoria, además de romper el scroll (habría que envolver el `Column` en un `verticalScroll` manual, que tampoco virtualiza nada).

`LazyColumn`/`LazyRow` resuelven esto con virtualización: componen solo lo visible en viewport (más un pequeño buffer), y a medida que el usuario scrollea, los ítems que salen de pantalla se destruyen y los que entran se componen recién en ese momento. Es el mismo problema que resuelve un `RecyclerView` en Android View clásico, pero declarativo.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayersList(
    players: List<Player>,
    onPlayerClick: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(
        modifier = modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = players,
            key = { player -> player.id } // clave estable, no el índice
        ) { player ->
            PlayerCard(
                player = player,
                onClick = onPlayerClick
            )
        }
    }
}
```

`items(items = players, key = { it.id })` le dice a `LazyColumn` cómo identificar cada elemento de forma única y estable. `contentPadding` (a diferencia de un `Modifier.padding()`) agrega espacio alrededor del contenido scrolleable sin afectar el área de scroll en sí — el primer y último ítem quedan con aire, pero el usuario puede seguir scrolleando "por encima" de ese padding.

## 4. Matriz de criterio

**`LazyColumn`/`LazyRow` vs `Column`/`Row` + scroll manual**
- Usar Lazy cuando: la cantidad de ítems es grande, desconocida de antemano, o proviene de una fuente de datos que puede crecer (una lista de partidas, un historial).
- Usar `Column`/`Row` simple cuando: la cantidad de ítems es fija y chica (un formulario de 5 campos, una fila de 3 botones de acción) — el overhead del DSL de Lazy no se justifica.
- Trade-off: Lazy gana performance y memoria a cambio de un API más rígido (no podés simplemente poner composables hijos directos, tenés que usar el scope `items`/`item`).

**`key` en `items()`**
- Usar cuando: siempre que sea posible — usar un identificador estable del dato (`player.id`), nunca el índice de la lista.
- NO usar el índice como key cuando: la lista puede reordenarse, insertar o eliminar elementos en el medio — si usás el índice, Compose puede confundir qué composable corresponde a qué dato tras un cambio, causando que el estado interno de un ítem (ej: un `remember` local, una animación en curso) "salte" al ítem incorrecto.
- Trade-off: omitir `key` funciona "por ahora" en listas estáticas, pero es una bomba de tiempo en cuanto la lista se vuelve dinámica — mejor incluirlo siempre por hábito, incluso cuando hoy no parece necesario.

**`itemsIndexed`**
- Usar cuando: necesitás tanto el índice como el valor (ej: mostrar "Jugador #3" o aplicar lógica de alternancia de color por fila).
- NO usar cuando: el índice no aporta nada al ítem en sí — agregarlo sin necesidad es ruido.

## 5. Caso trampa

```kotlin
// Lista de rondas, reordenable por el usuario (drag & drop)
LazyColumn {
    itemsIndexed(rounds) { index, round ->
        RoundCard(round = round, position = index)
        // sin key explícita
    }
}
```

La trampa: sin `key`, Compose usa la posición en la lista como identidad implícita del composable. Si el usuario reordena las rondas (drag & drop, o simplemente se inserta una ronda nueva en el medio), Compose no sabe que "la ronda que estaba en la posición 2 ahora está en la posición 4" — asume que el composable en la posición 2 simplemente cambió de datos. Cualquier estado local de ese composable (por ejemplo, un `remember { mutableStateOf(false) }` que controla si la card está expandida) queda pegado a la *posición*, no al dato — resultando en cards que aparecen expandidas o colapsadas de forma incorrecta después de reordenar, aunque el dato mostrado sea el correcto. La solución es siempre proveer un `key` basado en un identificador estable del dato (`round.id`), nunca en el índice, especialmente en listas que pueden reordenarse o mutar.

## 6. Conexión con arquitectura real

En Timbax, la lista de jugadores o el historial de partidas que expone `PlayersState.players` (una `List<Player>` que viene del `ViewModel`, alimentada en última instancia por `PlayerRepository`) se renderiza con `LazyColumn` usando `player.id` como `key` — el mismo `id` que ya es la identidad de dominio del modelo `Player` documentado en `02_domain/model.md`. Esto no es casualidad: la clave de identidad que definiste en el dominio (para `equals`/`copy` en la `data class`) es naturalmente la misma que Compose necesita para trackear identidad de UI — reforzando que el modelo de dominio bien diseñado se aprovecha en capas que ni siquiera lo conocen directamente.