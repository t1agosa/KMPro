# compositionlocal.md

## 1. Qué es

`CompositionLocal` es un mecanismo para pasar datos de forma **implícita** a través del árbol de composables, sin necesidad de declararlos como parámetro explícito en cada función intermedia del árbol. Se declara con `compositionLocalOf { valorPorDefecto }` o `staticCompositionLocalOf { valorPorDefecto }`, se "provee" un valor concreto con `CompositionLocalProvider(MiLocal provides valor) { contenido }`, y cualquier composable dentro de ese `contenido` puede leerlo con `MiLocal.current`, sin importar cuántos niveles de anidamiento haya en el medio.

Es, conceptualmente, similar a un `Context` de React o una inyección de dependencia acotada exclusivamente al árbol de composición — no es un mecanismo de DI general (para eso está Koin, ver `04_di`).

## 2. El problema que resuelve

Sin `CompositionLocal`, cualquier dato que necesite estar disponible en lo profundo de un árbol de composables (por ejemplo, los colores del tema) tendría que pasarse como parámetro explícito en **cada** función intermedia, incluso en las que no usan ese dato directamente, solo para poder pasarlo hacia sus hijos ("prop drilling"). Con un árbol de UI de 5-6 niveles de profundidad, eso ensucia la firma de funciones que conceptualmente no deberían saber nada sobre theming.

`CompositionLocal` resuelve esto haciendo que el dato "atraviese" el árbol de forma implícita: se provee una vez arriba, y se lee donde se necesite, sin que los niveles intermedios tengan que reenviarlo manualmente. `MaterialTheme` (documentado en `theming_material3.md`) está construido internamente sobre este mecanismo — es el ejemplo canónico de uso correcto.

## 3. Ejemplo mínimo comentado

```kotlin
// 1. Declaración: un CompositionLocal con valor por defecto
val LocalGameConfig = staticCompositionLocalOf<GameConfig> {
    error("No hay GameConfig provisto") // fuerza a que siempre se provea explícitamente
}

// 2. Provisión: en algún punto alto del árbol (ej: raíz de una pantalla de juego)
@Composable
fun GameScreen(config: GameConfig, content: @Composable () -> Unit) {
    CompositionLocalProvider(LocalGameConfig provides config) {
        content()
    }
}

// 3. Lectura: en cualquier composable hijo, sin importar la profundidad
@Composable
fun MaxPlayersLabel() {
    val config = LocalGameConfig.current // no necesitó recibirlo como parámetro
    Text("Máximo: ${config.maxPlayers} jugadores")
}
```

`MaxPlayersLabel` puede estar anidado 5 niveles debajo de `GameScreen` y ninguno de esos niveles intermedios necesitó declarar `config: GameConfig` en su firma solo para reenviarlo — cada uno se mantiene enfocado en su propia responsabilidad.

## 4. Matriz de criterio

**`compositionLocalOf` vs `staticCompositionLocalOf`**
- Usar `compositionLocalOf` cuando: el valor puede cambiar durante la vida de la composición y necesitás que Compose recomponga de forma granular solo los composables que leen `.current` — tiene un costo de tracking en runtime, pero permite cambios dinámicos correctamente.
- Usar `staticCompositionLocalOf` cuando: el valor es efectivamente estático dentro de esa porción del árbol (no va a cambiar mientras el `CompositionLocalProvider` esté activo, como el tema o la configuración de un juego ya iniciado) — es más performante porque Compose no necesita trackear cambios, pero si el valor cambia igual, Compose recompone **todo** el subárbol bajo ese `Provider`, no solo lo que lee `.current`.
- Trade-off: `staticCompositionLocalOf` mal usado con un valor que sí cambia frecuentemente produce recomposiciones mucho más amplias de lo necesario — es la contracara de la granularidad que documenta `recomposicion.md`.

**`CompositionLocal` vs parámetro explícito / vs `ViewModel`+DI**
- Usar `CompositionLocal` cuando: el dato es verdaderamente transversal a toda o gran parte de la UI, y no forma parte de la lógica de negocio — tema visual (colores, tipografía), configuración de layout (densidad, dirección de texto LTR/RTL), o el propio scope de navegación.
- NO usar cuando: el dato es un caso de uso, una regla de negocio, o algo que un `ViewModel` debería resolver — usar `CompositionLocal` para "esquivar" pasar un parámetro de negocio explícito dificulta rastrear de dónde viene un valor, porque cualquier composable en cualquier profundidad podría estar leyéndolo sin que se note en ninguna firma de función.
- Trade-off: gana comodidad y desacopla la firma de funciones intermedias, pero pierde trazabilidad explícita — es exactamente el trade-off que documenta la fuente original del repo ("no abusar: útil para datos verdaderamente transversales, pero para lógica de negocio o navegación se prefiere pasar explícito o vía ViewModel/DI").

**Valor por defecto con `error(...)` vs un valor "razonable"**
- Usar `error("mensaje")` como default cuando: el `CompositionLocal` **debe** ser provisto siempre antes de usarse, y preferís un crash inmediato y descriptivo en desarrollo antes que un valor incorrecto silencioso en producción.
- Usar un valor por defecto "razonable" cuando: existe un fallback genuinamente válido si nadie provee el valor (por ejemplo, `LocalContentColor` en Compose usa negro/blanco según el tema como default razonable).

## 5. Caso trampa

```kotlin
val LocalSelectedPlayerId = compositionLocalOf<String?> { null }

@Composable
fun TournamentScreen(viewModel: TournamentViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    CompositionLocalProvider(LocalSelectedPlayerId provides state.selectedPlayerId) {
        PlayersGrid()
    }
}

@Composable
fun PlayerCell(player: Player) {
    val selectedId = LocalSelectedPlayerId.current
    val isSelected = player.id == selectedId

    Box(modifier = Modifier.background(if (isSelected) Color.Yellow else Color.Transparent)) {
        Text(player.name)
    }

    Button(onClick = {
        // ¿cómo notifico al ViewModel que se seleccionó este jugador?
        // LocalSelectedPlayerId es de solo lectura (.current), no tiene "setter"
    }) { Text("Seleccionar") }
}
```

La trampa: `CompositionLocal` es un canal de **una sola dirección** — de arriba hacia abajo. Es tentador usarlo para "evitar pasar `onPlayerSelected` como parámetro por 3 niveles", pero `.current` solo permite leer, nunca escribir de vuelta hacia el `ViewModel`. El resultado es un diseño a medio camino: la lectura del estado seleccionado quedó desacoplada (cómoda), pero la escritura (el callback `onPlayerSelected`) sigue necesitando pasarse explícitamente como parámetro de todas formas — perdiendo gran parte del beneficio, y agregando una fuente de datos adicional (`CompositionLocal`) que hay que rastrear mentalmente además del flujo normal de `Event`/callback. La corrección más simple y consistente con MVI es hoistear `selectedPlayerId` y `onPlayerSelected` como parámetros explícitos, igual que cualquier otro estado (ver `composables_y_state_hoisting.md`) — `CompositionLocal` no es un atajo general para "estado compartido", es específicamente para datos de solo lectura, verdaderamente transversales.

## 6. Conexión con arquitectura real

En Timbax, el único uso legítimo de `CompositionLocal` hoy es indirecto, a través de `MaterialTheme` (`theming_material3.md`) — no hay (ni debería haber) `CompositionLocal` propios para estado de negocio como jugadores, puntajes o eventos de partida. Esa es una decisión de arquitectura deliberada: todo lo que tiene relevancia de negocio viaja por el canal explícito de MVI (`State` hacia abajo, `Event` hacia arriba, documentado en `06_presentation_mvi`), y `CompositionLocal` queda reservado exclusivamente para lo verdaderamente transversal y de solo lectura — tema, densidad, dirección de layout. Si en algún momento aparece la tentación de crear un `CompositionLocal` propio para algo de Timbax, la primera pregunta debería ser: "¿esto es tema/config visual, o es estado de negocio disfrazado de conveniencia?"