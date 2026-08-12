# effects_guia_completa.md

## 1. Qué es

Las side-effect APIs son el conjunto de funciones de Compose diseñadas para ejecutar código que "sale" del modelo puro de composición — llamadas de red, suscripciones a listeners, animaciones, notificar a librerías externas, lanzar coroutines desde un callback — de forma controlada y atada al ciclo de vida de la composición, en vez de ejecutarse directamente y sin control dentro del cuerpo de un `@Composable`.

Las 8 APIs que cubren prácticamente todos los casos reales son: `LaunchedEffect`, `rememberCoroutineScope`, `rememberUpdatedState`, `DisposableEffect`, `SideEffect`, `produceState`, `derivedStateOf` y `snapshotFlow`. Cada una resuelve un tipo distinto de efecto secundario; elegir la incorrecta no siempre rompe la compilación, pero sí produce comportamiento erróneo, fugas de recursos, o bugs de "closure vieja" difíciles de detectar a simple vista.

## 2. El problema que resuelve

Un `@Composable` idealmente es una función pura (documentado en `composables_y_state_hoisting.md`): mismo `State`, mismo resultado visual. El cuerpo de esa función puede ejecutarse **muchas veces** por segundo durante recomposición — así que cualquier efecto secundario escrito directamente en el cuerpo (una llamada de red, un `println` de analytics, suscribirse a un listener) se dispararía repetidamente, sin control, cada vez que Compose decide recomponer por cualquier motivo.

Además, Compose introduce un problema adicional que no existe en código imperativo: coroutines que solo pueden lanzarse *dentro* del cuerpo de un composable (`LaunchedEffect`), vs. coroutines que necesitan lanzarse *desde un callback* como `onClick` (donde `LaunchedEffect` no se puede usar). Las 8 APIs de esta guía cubren, entre todas, cada combinación posible de "cuándo se dispara" y "qué tipo de trabajo hace".

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayerDetailScreen(playerId: String, viewModel: PlayerDetailViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val scope = rememberCoroutineScope() // ver punto 2 más abajo
    val snackbarHostState = remember { SnackbarHostState() }

    // 1. LaunchedEffect: dispara una carga de datos al entrar, o si cambia playerId
    LaunchedEffect(playerId) {
        viewModel.onEvent(PlayerDetailEvent.OnScreenOpened(playerId))
    }

    // 2. rememberCoroutineScope: para lanzar coroutines desde un callback (onClick),
    // donde NO se puede declarar un LaunchedEffect directamente
    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("Guardado")
        }
    }) { Text("Guardar") }

    // 3. rememberUpdatedState: evita que un efecto de larga duración use un valor "viejo"
    val currentPlayerId by rememberUpdatedState(playerId)
    LaunchedEffect(Unit) { // key fija a propósito, ver Caso trampa
        while (true) {
            delay(30_000)
            viewModel.onEvent(PlayerDetailEvent.OnHeartbeat(currentPlayerId)) // siempre actualizado
        }
    }

    // 4. DisposableEffect: se suscribe a un listener externo y se desuscribe al salir
    DisposableEffect(Unit) {
        val listener = ConnectivityListener { isOnline ->
            viewModel.onEvent(PlayerDetailEvent.OnConnectivityChanged(isOnline))
        }
        listener.register()
        onDispose { listener.unregister() } // limpieza obligatoria
    }

    // 5. SideEffect: publica el score actual hacia una librería de analytics no-Compose
    SideEffect {
        AnalyticsTracker.setCurrentScore(state.player?.score ?: 0)
    }

    // 6. produceState: adapta una fuente asincrónica externa a un State observable
    val avatarBitmap by produceState<Bitmap?>(initialValue = null, playerId) {
        value = loadAvatarBitmap(playerId) // suspend fun externa
    }

    // 7. derivedStateOf: recalcula solo cuando el valor derivado realmente cambia
    val isHighScore by remember {
        derivedStateOf { (state.player?.score ?: 0) > 100 }
    }

    // 8. snapshotFlow: convierte lecturas de State de Compose en un Flow real
    val listState = rememberLazyListState()
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                viewModel.onEvent(PlayerDetailEvent.OnScrollPositionChanged(index))
            }
    }

    PlayerDetailContent(state = state, avatarBitmap = avatarBitmap, isHighScore = isHighScore)
}
```

Cada API resuelve un problema distinto: `LaunchedEffect` dispara trabajo asincrónico controlado por una key; `rememberCoroutineScope` habilita lanzar coroutines desde callbacks; `rememberUpdatedState` evita closures viejas en efectos de larga duración; `DisposableEffect` garantiza limpieza; `SideEffect` publica hacia afuera sin lanzar coroutines; `produceState` convierte algo externo en `State`; `derivedStateOf` evita recomposiciones innecesarias de un valor calculado; `snapshotFlow` tiende un puente entre el mundo de `State` de Compose y el mundo de operadores de `Flow`.

## 4. Matriz de criterio

**`LaunchedEffect(key)`**
- Usar cuando: necesitás lanzar una coroutine ligada al ciclo de vida de la composición — cargar datos al entrar a la pantalla (`LaunchedEffect(Unit)`), o relanzar la carga cuando cambia un parámetro (`LaunchedEffect(playerId)`).
- NO usar cuando: el trabajo no es realmente asincrónico/suspendible (ahí `SideEffect`), o cuando necesitás lanzar la coroutine desde un callback como `onClick` (ahí `rememberCoroutineScope`, porque `LaunchedEffect` solo puede declararse en el cuerpo del composable, nunca dentro de un lambda de evento).
- Trade-off: elegir mal la `key` es la fuente de bugs más común (ver Caso trampa 1) — la key determina cuándo se cancela el efecto anterior y se relanza uno nuevo.

**`rememberCoroutineScope`**
- Usar cuando: necesitás lanzar una coroutine en respuesta a un evento puntual del usuario (`onClick`, `onDismiss`) que no está atado a la composición en sí, sino a una acción discreta — el ejemplo clásico es mostrar un `Snackbar` desde un botón.
- NO usar cuando: el trabajo debería dispararse automáticamente al entrar a composición o cuando cambia una key — eso es exactamente el caso de uso de `LaunchedEffect`, no de `rememberCoroutineScope`.
- Trade-off: el scope obtenido vive tanto como el composable que lo creó (se cancela cuando ese composable sale de composición) — hay que asegurarse de que el callback donde se usa tenga sentido dentro de esa vida útil.

**`rememberUpdatedState`**
- Usar cuando: tenés un efecto de larga duración con una `key` fija a propósito (por ejemplo, `LaunchedEffect(Unit)` con un loop infinito que no debería reiniciarse) pero ese efecto necesita leer un valor que sí cambia con el tiempo (como `playerId` en el ejemplo, o un callback `onTimeout` recibido por parámetro).
- NO usar cuando: podés simplemente incluir el valor como `key` del efecto y dejar que se relance — es más simple y evita el patrón intermedio. `rememberUpdatedState` se reserva para cuando relanzar el efecto completo sería costoso o incorrecto (por ejemplo, un timer que no debe resetearse cada vez que cambia un dato no relacionado).
- Trade-off: agrega una capa de indirección (`by rememberUpdatedState(valor)`) que hay que entender explícitamente — mal usado, alguien puede "arreglar" un bug de closure vieja poniendo la key directamente y accidentalmente reiniciando un efecto que no debía reiniciarse.

**`DisposableEffect(key)`**
- Usar cuando: el efecto necesita una limpieza explícita al salir de composición — suscribirse/desuscribirse a un listener, un observer del sistema operativo, un callback nativo.
- NO usar cuando: no hay nada que limpiar — si no necesitás `onDispose`, probablemente te alcance con `LaunchedEffect`.
- Trade-off: obliga a terminar siempre con `onDispose { }` (el compilador lo exige) — más verboso, pero evita fugas de recursos/memoria por listeners que nunca se desregistran.

**`SideEffect`**
- Usar cuando: necesitás publicar un valor de Compose hacia código no-Compose (analytics, un View nativo interop) sin lanzar una coroutine — se ejecuta después de cada recomposición exitosa.
- NO usar cuando: el trabajo es asincrónico o necesita limpieza — para eso están `LaunchedEffect`/`DisposableEffect` respectivamente.

**`produceState`**
- Usar cuando: necesitás adaptar una fuente externa (una `suspend fun`, un callback, un `Flow` de una librería que no es nativamente de Compose) a un `State<T>` que Compose pueda leer y recomponer en base a él.
- NO usar cuando: ya tenés un `StateFlow` del `ViewModel` — ahí corresponde `collectAsStateWithLifecycle()` directamente (ver `collect_stateflow_en_compose.md`), no reinventar el mecanismo con `produceState`.

**`derivedStateOf`**
- Usar cuando: calculás un valor derivado de otro `State` que cambia con más frecuencia de la que el resultado derivado realmente cambia (el ejemplo clásico: un booleano que depende de la posición de scroll, que cambia en cada pixel pero el booleano solo en umbrales puntuales).
- NO usar cuando: el cálculo derivado es trivial y barato, y cambia al mismo ritmo que su fuente — ahí `derivedStateOf` agrega complejidad sin beneficio real de performance.
- Trade-off: siempre se envuelve en `remember { derivedStateOf { ... } }` — omitir el `remember` externo recrea el `derivedStateOf` en cada recomposición, anulando por completo su propósito.

**`snapshotFlow`**
- Usar cuando: necesitás aplicar operadores de `Flow` (`debounce`, `distinctUntilChanged`, `filter`, `map`) sobre cambios de un `State` de Compose — por ejemplo, reaccionar a la posición de scroll de una `LazyColumn` solo después de que el usuario dejó de scrollear.
- NO usar cuando: solo necesitás leer el valor actual del `State` sin transformarlo con operadores de `Flow` — ahí simplemente se lee el `State` directamente, `snapshotFlow` sería una complicación innecesaria.
- Trade-off: tiende un puente entre dos mundos (Snapshot state de Compose y `Flow` de coroutines) — es potente pero es la API menos intuitiva de las 8 si no se entendió primero qué es un `Flow` (ver `08_flow/flow_basico.md`).

## 5. Caso trampa

**Trampa 1 — `key` incorrecta en `LaunchedEffect`:**

```kotlin
@Composable
fun PlayerDetailScreen(playerId: String, viewModel: PlayerDetailViewModel) {
    LaunchedEffect(Unit) { // key fija, "solo una vez"
        viewModel.onEvent(PlayerDetailEvent.OnScreenOpened(playerId))
    }
}
```

`LaunchedEffect(Unit)` se lanza una sola vez y nunca más, sin importar qué pase con `playerId` después. Si el composable persiste entre cambios de `playerId` (navegación que reutiliza la instancia), la pantalla queda mostrando datos del jugador equivocado. La corrección es `LaunchedEffect(playerId)`.

**Trampa 2 — closure vieja en un efecto de larga duración (la razón de ser de `rememberUpdatedState`):**

```kotlin
@Composable
fun PlayerDetailScreen(playerId: String, viewModel: PlayerDetailViewModel) {
    // loop infinito de heartbeat, con key fija a propósito (no queremos reiniciarlo)
    LaunchedEffect(Unit) {
        while (true) {
            delay(30_000)
            viewModel.onEvent(PlayerDetailEvent.OnHeartbeat(playerId)) // <- BUG
        }
    }
}
```

La trampa es más sutil que la anterior: acá la intención de usar `LaunchedEffect(Unit)` es correcta (no querés reiniciar el timer cada vez que cambia `playerId`), pero el lambda de la coroutine **capturó el valor de `playerId` en el momento en que se lanzó** — como es una `key` fija, ese lambda nunca se vuelve a crear, así que sigue usando el `playerId` original para siempre, aunque el composable haya recompuesto muchas veces con un `playerId` distinto. El bug es invisible en el código: compila perfecto, y en desarrollo con navegación simple puede no notarse. La solución es exactamente `rememberUpdatedState`: envolver `playerId` para que el efecto de larga duración siempre lea la versión más reciente sin necesitar reiniciarse.

## 6. Conexión con arquitectura real

En Timbax, `LaunchedEffect(Unit)` es el mecanismo estándar para disparar la carga inicial de datos al entrar a una pantalla — típicamente emitiendo un `Event` como `OnScreenOpened` hacia el `ViewModel`, que a su vez llama al `UseCase` correspondiente (ver `06_presentation_mvi/viewmodel.md`). Es también el punto de entrada para **colectar `Effect`** del `ViewModel` (ver `08_flow/sharedflow_channel.md`): un `LaunchedEffect(Unit) { viewModel.effect.collect { ... } }` es lo que conecta el `SharedFlow(replay=0)` de Effects con acciones reales de UI como navegar o mostrar un snackbar — el archivo `navegacion.md` retoma este mismo patrón para cerrar el loop completo Event → Effect → navigate.

`rememberCoroutineScope` entra en juego específicamente cuando un `Snackbar` de confirmación se dispara desde un botón (`onClick`), en vez de como reacción a un `Effect` del `ViewModel` — un caso de UI puramente local que no necesita ida y vuelta al `ViewModel`. `snapshotFlow` es la herramienta correcta si en el futuro Timbax necesita, por ejemplo, guardar la posición de scroll de un historial largo de partidas solo después de que el usuario deja de scrollear, sin disparar un `Event` en cada pixel.