# layouts_basicos.md

## 1. Qué es

Los layouts básicos son los tres composables fundamentales que definen cómo se organizan otros composables en el espacio: `Column` (apila verticalmente), `Row` (apila horizontalmente) y `Box` (apila en capas, unos sobre otros). A esto se suma `Scaffold`, que no es un layout de posicionamiento sino un composable estructural: define el esqueleto estándar de una pantalla (top bar, contenido, bottom bar, FAB) y calcula automáticamente el padding necesario para que el contenido no quede tapado por esas piezas.

Todo layout más complejo en Compose (una lista, una card, una pantalla completa) es, en el fondo, una composición de estos tres bloques.

## 2. El problema que resuelve

En sistemas de UI más viejos (XML de Android View, por ejemplo), el posicionamiento se declaraba con atributos dispersos por muchas líneas (`layout_below`, `layout_toEndOf`, `layout_gravity`) y era fácil terminar con jerarquías de `RelativeLayout` anidados difíciles de leer. Compose resuelve esto con tres primitivas simples y componibles: si necesitás algo vertical, es un `Column`; si es horizontal, un `Row`; si es superposición, un `Box`. La complejidad de un layout real surge de anidar estos tres de forma clara, no de aprender un cuarto o quinto tipo de contenedor.

`Scaffold` resuelve un problema distinto: sin él, cada pantalla tendría que calcular a mano cuánto espacio ocupa una `TopAppBar` o un `FloatingActionButton` para no tapar el contenido debajo. `Scaffold` expone ese cálculo como un parámetro (`innerPadding`) que estás obligado a usar.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayersScreen(
    state: PlayersState,
    onEvent: (PlayersEvent) -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Jugadores") })
        }
    ) { innerPadding ->
        // innerPadding viene de Scaffold: asegura que el contenido
        // no quede tapado por la TopAppBar
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(innerPadding),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            state.players.forEach { player ->
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween
                ) {
                    Text(player.name)

                    // Box permite superponer un badge sobre el score
                    Box(contentAlignment = Alignment.TopEnd) {
                        Text("${player.score}")
                        if (player.score > 100) {
                            Badge()
                        }
                    }
                }
            }
        }
    }
}
```

`Column` ordena la lista de jugadores verticalmente. Cada jugador es un `Row` que pone nombre y score en extremos opuestos (`SpaceBetween`). El `Box` alrededor del score permite dibujar un `Badge` superpuesto solo cuando corresponde — algo que ni `Column` ni `Row` pueden hacer, porque ninguno de los dos apila en el eje Z.

## 4. Matriz de criterio

**`Column` / `Row`**
- Usar cuando: el contenido tiene un orden secuencial natural (una lista de campos, botones en fila).
- NO usar cuando: el contenido es realmente una lista larga o desconocida en tamaño — ahí corresponde `LazyColumn`/`LazyRow` (ver `listas_lazy.md`), porque `Column` compone TODOS los hijos de una, aunque no estén visibles en pantalla.
- Trade-off: simples y predecibles, pero sin virtualización — usarlos para listas grandes es un error de performance real, no cosmético.

**`Box`**
- Usar cuando: necesitás superposición real (badges, overlays de loading sobre contenido, fondos con contenido encima) o cuando necesitás un contenedor con un solo hijo que se puede alinear (`contentAlignment`).
- NO usar cuando: lo usás como reemplazo perezoso de `Column`/`Row` porque "total, con Modifier lo acomodo" — eso ensucia la semántica del layout y complica el mantenimiento.
- Trade-off: máxima flexibilidad de posicionamiento, pero ninguna estructura implícita — vos sos responsable de que el `Alignment` de cada hijo tenga sentido.

**`Scaffold`**
- Usar cuando: es la raíz de una pantalla completa (siempre que haya top bar, bottom bar, snackbar host o FAB).
- NO usar cuando: es un componente interno de una pantalla (una card, un item de lista) — `Scaffold` es un patrón a nivel pantalla, no un layout de propósito general.
- Trade-off: te obliga a manejar el `innerPadding` correctamente; ignorarlo es el error más común (ver Caso trampa).

## 5. Caso trampa

Ignorar el `innerPadding` que provee `Scaffold`:

```kotlin
// MAL: el Column ignora innerPadding
Scaffold(topBar = { TopAppBar(title = { Text("Jugadores") }) }) { innerPadding ->
    Column(modifier = Modifier.fillMaxSize()) {
        // el primer item queda parcialmente tapado por la TopAppBar
    }
}
```

La trampa es que esto *compila* y *corre* sin errores — visualmente puede incluso parecer correcto en un preview chico, pero en un dispositivo real el contenido queda parcialmente oculto detrás de la `TopAppBar`. `Scaffold` no fuerza el uso del padding en tiempo de compilación; es responsabilidad de quien escribe el composable aplicarlo (`.padding(innerPadding)`). Es un error silencioso, no uno que el compilador te señale.

## 6. Conexión con arquitectura real

En Timbax, `PlayersScreen` es un composable "tonto" (stateless) que recibe `PlayersState` y emite `PlayersEvent` — el mismo principio de **state hoisting** que vamos a documentar en `composables_y_state_hoisting.md`. Los layouts básicos son la capa más externa de esa idea: `Scaffold` define el marco de la pantalla, `Column`/`Row`/`Box` organizan el contenido derivado de `state.players`, pero ninguno de los tres decide *qué* mostrar — eso ya vino resuelto desde el `ViewModel` en forma de `State` inmutable. El layout solo traduce datos ya calculados a posiciones en pantalla.