# recomposicion.md

## 1. Qué es

Recomposición es el proceso por el cual Compose vuelve a ejecutar una función `@Composable` cuando alguno de los valores de `State` que esa función lee cambia. No es "redibujar toda la pantalla desde cero": Compose lleva un registro fino de qué composable leyó qué `State`, y solo vuelve a ejecutar (recompone) las funciones que efectivamente dependen del valor que cambió — el resto del árbol se saltea (**recomposition skipping**).

## 2. El problema que resuelve

En un modelo de UI imperativo (Android View clásico), cuando un dato cambia, el desarrollador tiene que buscar manualmente cada `View` afectada y llamar a `setText()`, `setVisibility()`, etc. — es responsabilidad humana mantener la UI sincronizada con el estado, y es una fuente constante de bugs ("me olvidé de actualizar ese label").

Compose invierte el problema: la UI se declara como una función del `State` (`@Composable fun Screen(state: State)`), y el framework se encarga de detectar qué cambió y volver a ejecutar solo esa parte. El desarrollador nunca escribe código imperativo de "actualizar esto"; solo describe "así se ve dado este estado", y Compose decide cuándo y qué recomponer.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayersScreen(state: PlayersState) {
    Column {
        Header(title = "Jugadores") // no lee ningún State que cambie -> nunca recompone

        Text("Total: ${state.players.size}") // lee state.players -> recompone si cambia

        PlayerScoreDisplay(score = state.selectedPlayerScore) // lee solo el score
    }
}

@Composable
fun PlayerScoreDisplay(score: Int) {
    // Este composable SOLO recompone si "score" cambia,
    // sin importar que otras partes de PlayersState hayan cambiado
    Text("Puntaje: $score")
}
```

Si `state.players` cambia pero `state.selectedPlayerScore` no, Compose recompone `Text("Total: ...")` pero **saltea** `PlayerScoreDisplay` — porque ese composable nunca leyó `state.players`, solo `score`. `Header` nunca recompone en absoluto, porque no lee ningún valor de `State`.

## 4. Matriz de criterio

**Granularidad del `State` que se lee**
- Usar cuando: se pasa exactamente el valor que un composable necesita (`score: Int`), en vez de todo el objeto contenedor (`state: PlayersState`) — esto maximiza cuánto puede skipear Compose.
- NO pasar el objeto completo cuando: el composable solo necesita un campo — pasar todo `PlayersState` hace que ese composable "lea" (a ojos de Compose) el objeto entero, y cualquier cambio en cualquier campo fuerza su recomposición aunque el campo que realmente usa no haya cambiado.
- Trade-off: pasar campos individuales es más verboso en la firma de la función, pero es lo que realmente habilita el recomposition skipping; pasar el objeto completo es más cómodo de escribir pero sacrifica performance.

**Composable sin parámetros de `State` (como `Header` arriba)**
- Usar cuando: el contenido es estático o depende solo de constantes/recursos — nunca necesita recomponer.
- Confirmar que efectivamente no lee ningún `State` — si por error termina leyendo algo (por ejemplo, una variable de un scope externo que sí cambia), pierde esa garantía sin que el compilador avise.

**Cuándo preocuparse por recomposición innecesaria**
- Investigar cuando: hay jank visible (frames perdidos), listas largas que se sienten lentas al scrollear, o composables pesados (con cálculos costosos en el cuerpo) que recomponen más seguido de lo esperado.
- NO optimizar prematuramente cuando: la pantalla es simple y no hay síntomas de performance — perseguir el skipping perfecto en toda la app sin evidencia de un problema real es tiempo mal invertido; para eso existen herramientas como el Layout Inspector de Android Studio, que muestra conteos de recomposición reales.

## 5. Caso trampa

```kotlin
@Composable
fun PlayersScreen(viewModel: PlayersViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    Column {
        PlayersHeader(state = state)      // recibe el objeto completo
        PlayersList(state = state)        // recibe el objeto completo
        SelectedPlayerCard(state = state) // recibe el objeto completo
    }
}
```

La trampa: parece prolijo — cada composable recibe "el estado de la pantalla" y listo. Pero como los tres reciben el mismo `PlayersState` completo, cualquier cambio en *cualquier* campo (por ejemplo, si solo cambia `state.isLoading`) hace que Compose deba recomponer los tres, aunque `PlayersHeader` solo use `state.title` y `SelectedPlayerCard` solo use `state.selectedPlayer`. La UI sigue viéndose correcta — el bug no es visual, es de performance silenciosa: en una pantalla simple no se nota, pero en una pantalla con composables pesados o listas largas, cada cambio de un campo aislado del `State` dispara recomposiciones innecesarias en cascada. La corrección es desestructurar en el nivel más alto posible y pasar campos puntuales: `PlayersHeader(title = state.title)`, `SelectedPlayerCard(player = state.selectedPlayer)`.

## 6. Conexión con arquitectura real

En Timbax, cada vez que el `ViewModel` hace `_state.update { it.copy(...) }`, se emite un `PlayersState` completo nuevo por el `StateFlow` — eso es correcto y necesario para MVI (ver `08_flow/stateflow.md`: "State: única fuente de verdad"). Pero **cómo se consume** ese `State` en los composables de `ui` es una decisión distinta: aunque el `ViewModel` emita el objeto completo, la responsabilidad de la capa de UI es desestructurarlo en el punto de entrada (`PlayersScreen`) y pasar campos puntuales hacia abajo — así MVI mantiene su garantía de estado único y consistente en `presentation`, sin sacrificar la performance de recomposición en `ui`. Son dos capas con responsabilidades distintas, aunque ambas toquen el mismo `State`.