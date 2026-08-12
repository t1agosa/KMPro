
# collect_stateflow_en_compose.md

## 1. Qué es

`collectAsStateWithLifecycle()` es la función que conecta un `StateFlow` del `ViewModel` con Compose, convirtiéndolo en un `State<T>` que un composable puede leer y ante cuyos cambios puede recomponer. Es la puerta de entrada estándar por la cual el `State` de MVI (documentado en `06_presentation_mvi` y `08_flow/stateflow.md`) llega finalmente a la UI.

Existe también `collectAsState()` (sin el sufijo `WithLifecycle`), una versión más simple pero con una diferencia de comportamiento importante que determina cuál corresponde usar.

## 2. El problema que resuelve

Un `StateFlow` es un stream de coroutines — no tiene, por sí mismo, ninguna noción de "la UI está visible" o "la app está en background". Si un composable colecta un `StateFlow` sin ningún control de ciclo de vida, sigue colectando (y potencialmente recomponiendo, o manteniendo trabajo activo aguas arriba) incluso cuando la app pasó a background y no hay nada visible en pantalla — desperdiciando batería y CPU, y en casos más severos, generando crashes por intentar actualizar UI que ya no está en un estado seguro para recibir cambios.

`collectAsStateWithLifecycle()` resuelve esto atando la colección al ciclo de vida real de la plataforma: pausa automáticamente cuando el composable no está en foreground, y la reanuda cuando vuelve — sin que el desarrollador tenga que gestionar esto manualmente.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayersScreen(viewModel: PlayersViewModel) {
    // recomendado: pausa la colección cuando la app no está en foreground
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column {
        if (state.isLoading) {
            CircularProgressIndicator()
        } else {
            PlayersList(players = state.players)
        }
    }
}
```

`by` aplica **delegated properties** (documentado en `Keywords_guide`, sección `by`): `state` se comporta como una variable normal (`state.isLoading`, `state.players`) aunque en realidad esté delegando su lectura a un `State<PlayersState>` por detrás. Cada vez que el `StateFlow` del `ViewModel` emite un nuevo `PlayersState`, Compose recompone automáticamente las partes de `PlayersScreen` que leen `state`.

## 4. Matriz de criterio

**`collectAsStateWithLifecycle()` vs `collectAsState()`**
- Usar `collectAsStateWithLifecycle()` cuando: siempre, en Android/producción — es la opción recomendada oficialmente para colectar flows atados a UI, porque respeta el ciclo de vida de la plataforma (pausa en background).
- Usar `collectAsState()` cuando: en contextos donde no aplica el concepto de "ciclo de vida de plataforma" de la misma forma (algunos targets no-Android de Compose Multiplatform, o en tests/previews simplificados) — pero en el código de producción de una app KMP con target Android, no hay buena razón para preferirlo sobre la versión con lifecycle.
- Trade-off: `collectAsStateWithLifecycle()` requiere la dependencia de `androidx.lifecycle` correspondiente para KMP — un costo de setup mínimo comparado con el beneficio de no desperdiciar recursos en background.

**Dónde llamar la colección (nivel de `Screen` vs nivel de composable hijo)**
- Usar cuando: la colección se hace **una sola vez**, en el composable más alto de la pantalla (`PlayersScreen`), y de ahí se desestructura y pasa como parámetros puntuales hacia los hijos — coincide con el criterio ya documentado en `recomposicion.md` sobre granularidad de `State`.
- NO usar cuando: cada composable hijo colecta el `StateFlow` del `ViewModel` de forma independiente — eso multiplica colecciones redundantes del mismo stream, y además rompe el patrón de composables stateless (`composables_y_state_hoisting.md`), porque cada hijo terminaría necesitando una referencia directa al `ViewModel` en vez de recibir datos por parámetro.

**`state.value` (acceso directo) vs `by` (delegated property)**
- Usar `by` cuando: siempre — es la forma idiomática, permite tratar `state` como si fuera el valor mismo (`state.players`) en vez de tener que escribir `state.value.players` en cada lectura.
- Evitar el acceso manual `.value` repetido — no es incorrecto, pero es más verboso sin ningún beneficio a cambio.

## 5. Caso trampa

```kotlin
@Composable
fun PlayerCard(player: Player) {
    Row {
        Text(player.name)
        FavoriteButton(playerId = player.id)
    }
}

@Composable
fun FavoriteButton(playerId: String, viewModel: FavoritesViewModel = koinInject()) {
    // cada card de la lista colecta su PROPIA copia del StateFlow completo
    val state by viewModel.state.collectAsStateWithLifecycle()
    val isFavorite = state.favoriteIds.contains(playerId)

    IconButton(onClick = { viewModel.onEvent(FavoritesEvent.OnToggle(playerId)) }) {
        Icon(if (isFavorite) Icons.Filled.Star else Icons.Outlined.Star, null)
    }
}
```

La trampa: si `FavoriteButton` se usa dentro de una `LazyColumn` con 50 jugadores, esto significa **50 colecciones independientes** del mismo `StateFlow` de `FavoritesViewModel` — cada una recomponiendo su propio `IconButton` de forma redundante cada vez que `state.favoriteIds` cambia, aunque solo un jugador haya cambiado de favorito. Funciona, no hay error, pero es ineficiente y además rompe la idea de "colectar una vez arriba, desestructurar hacia abajo": la fuente de verdad del favorito de cada jugador debería resolverse en el `PlayersScreen` (colectando `FavoritesViewModel` una sola vez ahí) y pasarse como un simple `Boolean isFavorite` a `FavoriteButton`, dejando a este último completamente stateless, sin necesitar conocer al `ViewModel` en absoluto.

## 6. Conexión con arquitectura real

En Timbax, `collectAsStateWithLifecycle()` se llama **una única vez** por pantalla, siempre en el composable raíz que recibe el `ViewModel` inyectado vía Koin (ver `04_di/koin_fundamentos_y_scopes.md`, scope `viewModel`) — nunca en composables hijos intermedios. Esa disciplina es lo que mantiene consistente toda la cadena documentada en `composables_y_state_hoisting.md`: `ViewModel` (única fuente de verdad) → `Screen` (única colección) → composables hijos (stateless, reciben solo lo que necesitan). Romper esa regla, aunque técnicamente "funcione", reintroduce fuentes de estado dispersas — exactamente el problema que MVI existe para prevenir.