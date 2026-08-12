# stability.md

## 1. Qué es

**Stability** es el criterio que usa el compilador de Compose para decidir si puede confiar en un tipo lo suficiente como para **skipear** la recomposición de un composable que lo recibe como parámetro. Un tipo es "estable" si Compose puede garantizar que, mientras sus propiedades públicas no cambien, dos instancias del mismo tipo producen el mismo resultado visual — y que Compose se va a enterar si efectivamente cambiaron (vía `equals()`).

Este análisis lo hace el **compiler plugin de Compose** en tiempo de compilación, no en runtime: cada tipo queda marcado internamente como estable o inestable antes de que la app corra.

## 2. El problema que resuelve

Recomposition skipping (documentado en `recomposicion.md`) depende de que Compose pueda comparar "¿los parámetros de este composable cambiaron respecto a la última vez?". Si un tipo es mutable de forma que Compose no puede verificar con seguridad (por ejemplo, una interfaz como `List<T>`, que en teoría podría ser una `MutableList` mutada por fuera sin que nadie se entere), Compose no puede confiar en que no cambió — y para estar seguro, **recompone igual**, incluso si en la práctica nada cambió.

Sin el concepto de stability, cada composable con un parámetro de tipo "dudoso" perdería automáticamente el beneficio de skipping, sin que el desarrollador tenga ninguna señal de por qué su pantalla recompone más de lo esperado.

## 3. Ejemplo mínimo comentado

```kotlin
// ESTABLE: data class con solo propiedades val de tipos estables
data class Player(
    val id: String,
    val name: String,
    val score: Int
)
// Compose puede confiar: si el equals() dice que no cambió, no cambió.

// INESTABLE: expone una List<T> (interfaz), no una implementación concreta inmutable
data class PlayersState(
    val players: List<Player>, // <- inestable a ojos del compilador
    val isLoading: Boolean
)

@Composable
fun PlayersList(players: List<Player>) {
    // Este composable NO puede skipear recomposición basándose solo en "players",
    // porque List<T> es una interfaz — podría ser una MutableList mutada por fuera
    LazyColumn {
        items(players, key = { it.id }) { player -> PlayerCard(player) }
    }
}
```

`Player` es estable porque todas sus propiedades (`String`, `Int`) son tipos estables y son `val`. `List<Player>`, en cambio, es la interfaz de Kotlin, no una implementación específica — Compose no puede garantizar en compile-time que la instancia real detrás de esa interfaz no vaya a mutar sin pasar por una nueva composición.

## 4. Matriz de criterio

**`List<T>` vs `ImmutableList<T>` (de `kotlinx.collections.immutable`)**
- Usar `ImmutableList<T>` cuando: el `State` que expone el `ViewModel` incluye colecciones, y te importa que Compose pueda skipear recomposición correctamente en listas grandes o pantallas con composables costosos.
- Seguir usando `List<T>` cuando: es un proyecto chico, sin síntomas de performance, donde agregar la dependencia extra no se justifica todavía.
- Trade-off: `ImmutableList` requiere agregar la librería `kotlinx.collections.immutable` y convertir explícitamente (`.toImmutableList()`) en el punto donde se construye el `State` — más fricción a cambio de una garantía real de skipping.

**`data class` con `val` vs `var`**
- Usar `val` siempre cuando: el `State` de MVI ya exige inmutabilidad por diseño (ver `06_presentation_mvi`) — esto además de ser correcto para MVI, es lo que hace que la `data class` sea estable.
- NO usar `var` en propiedades de un `State` — además de romper el patrón MVI de estado inmutable, un `var` hace que el compilador marque el tipo como inestable, porque ese campo podría mutar sin pasar por un nuevo `copy()`.

**Cuándo preocuparse por stability**
- Investigar cuando: hay evidencia real de recomposición excesiva (mismo criterio que en `recomposicion.md` — el Layout Inspector muestra conteos altos en composables que no deberían recomponer).
- NO reescribir todo el `State` a tipos inmutables "por las dudas" sin evidencia — es optimización, no corrección; el código sigue siendo funcionalmente correcto con `List<T>` normal, solo pierde una optimización de performance.

## 5. Caso trampa

```kotlin
data class PlayersState(
    val players: List<Player> = emptyList(),
    val isLoading: Boolean = false
)

@Composable
fun PlayersScreen(viewModel: PlayersViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // Solo cambia isLoading, players es "la misma lista" en términos de contenido
    PlayersList(players = state.players)
}
```

La trampa: intuitivamente, si `state.players` tiene el mismo contenido que antes (mismos jugadores, mismo orden), uno esperaría que `PlayersList` no recomponga cuando solo cambia `isLoading`. Pero como `List<Player>` es inestable a ojos del compilador, Compose **no puede confiar** en que la lista es "la misma" solo comparando referencias o incluso `equals()` con la garantía necesaria para skipear — así que recompone `PlayersList` en cada emisión del `StateFlow`, incluso cuando el cambio real fue en un campo completamente distinto del `State`. Esto pasa *incluso si Compose está haciendo bien su trabajo con el resto*: es específicamente la inestabilidad de `List<T>` la que rompe el skipping en este composable puntual, no un error de código. La corrección real es usar `ImmutableList<Player>` en el `State`, no reestructurar el resto de la arquitectura.

## 6. Conexión con arquitectura real

En Timbax, cada vez que `PlayersState` expone una colección (`players: List<Player>`, o un futuro `matchHistory: List<Round>`), la decisión de si usar `List<T>` simple o `ImmutableList<T>` es exactamente el tipo de decisión que documenta la Matriz de criterio de este archivo: no es "siempre usar Immutable" ni "nunca preocuparse", sino evaluarlo cuando la pantalla en cuestión (por ejemplo, un historial largo de partidas con `LazyColumn`, ver `listas_lazy.md`) muestra señales reales de recomposición innecesaria. Es la contracara práctica de `recomposicion.md`: ese archivo explica *qué* es el skipping, este explica *qué condiciones del tipo* lo habilitan o lo bloquean silenciosamente.