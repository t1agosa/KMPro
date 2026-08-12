# composables_y_state_hoisting.md

## 1. Qué es

Un `@Composable` es una función que describe una porción de UI a partir de datos de entrada, y que Compose puede volver a ejecutar (recomponer) cuando esos datos cambian. **State hoisting** es el patrón de "elevar" el estado desde el composable hijo hacia su padre: el hijo deja de guardar su propio estado interno y en su lugar recibe el valor actual (`value`) y una función para pedir que cambie (`onValueChange`), convirtiéndose en **stateless**.

Un composable stateless es, idealmente, una función pura: mismo input (mismo `State`), mismo resultado visual — sin memoria propia, sin efectos colaterales ocultos.

## 2. El problema que resuelve

Si cada composable guardara su propio estado interno (por ejemplo, un `Switch` que mantiene su `checked` con un `remember` propio), ese estado queda encerrado ahí — nadie fuera del composable puede leerlo, validarlo, ni decidir qué hacer con él. En una pantalla real, casi siempre alguien más necesita saber ese valor: el `ViewModel` para persistirlo, otro composable hermano para reaccionar a él, o una regla de negocio que depende de él.

State hoisting resuelve esto separando dos responsabilidades que suelen mezclarse: "cómo se ve/comporta la UI" (el composable) de "cuál es la fuente de verdad del dato" (quien lo hoistea, en última instancia el `ViewModel`). El composable se vuelve reusable (sirve para cualquier fuente de estado, no solo para un `remember` interno), testeable (podés testear su salida visual dado un `value` fijo, sin necesitar disparar estado interno oculto) y predecible.

## 3. Ejemplo mínimo comentado

```kotlin
// SIN hoisting: el composable "posee" su estado, nadie afuera puede leerlo ni controlarlo
@Composable
fun PlayerNameFieldStateful() {
    var name by remember { mutableStateOf("") }
    TextField(value = name, onValueChange = { name = it })
}

// CON hoisting: el estado vive afuera, el composable es una función pura de sus params
@Composable
fun PlayerNameField(
    name: String,
    onNameChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    TextField(
        value = name,
        onValueChange = onNameChange,
        modifier = modifier
    )
}

// Quien hoistea el estado: en este caso, directamente el StateFlow del ViewModel
@Composable
fun AddPlayerScreen(viewModel: AddPlayerViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    PlayerNameField(
        name = state.name,
        onNameChange = { viewModel.onEvent(AddPlayerEvent.OnNameChanged(it)) }
    )
}
```

`PlayerNameField` no sabe ni le importa de dónde viene `name` — podría ser un `remember` local en un preview, o el `State` real del `ViewModel` en producción. Esa independencia es exactamente lo que lo hace reusable.

## 4. Matriz de criterio

**Composable stateful (con `remember` propio) vs stateless (hoisted)**
- Usar stateful cuando: el estado es puramente de UI, efímero, y a nadie más le importa (ej: si un tooltip está expandido, la posición de scroll visual momentánea).
- Usar stateless (hoisted) cuando: el dato importa fuera del composable — se persiste, se valida, se comparte entre composables, o forma parte de una regla de negocio.
- Trade-off: hoistear todo "por las dudas" agrega verbosidad innecesaria a estado que genuinamente es solo de UI; no hoistear nada convierte al composable en una caja negra imposible de testear o reusar con otra fuente de datos.

**Patrón `value` + `onValueChange`**
- Usar cuando: es la convención estándar de Compose para cualquier composable con estado hoisted — mantenerla hace que el composable se sienta "nativo" y predecible para cualquiera que lea el código.
- NO inventar nombres de parámetros alternativos (`text`/`updateText`, `data`/`change`) sin razón — romper la convención obliga a quien lee el código a redescubrir el patrón en cada composable distinto.

**Nivel al que se hoistea el estado**
- Usar el nivel más bajo posible cuando: el estado no necesita salir de un composable padre inmediato (ej: si dos hijos hermanos comparten un estado, hoistealo al padre común, no directamente al `ViewModel`).
- Usar el `ViewModel` (vía `State`) cuando: el dato tiene relevancia de negocio, debe sobrevivir recomposición/rotación, o afecta a más de una sección de la pantalla.
- Trade-off: hoistear "demasiado alto, siempre al ViewModel" infla el `State` con cosas que son puramente de presentación visual local; hoistear "muy bajo, nunca al ViewModel" termina duplicando lógica en composables que deberían compartir una única fuente de verdad.

## 5. Caso trampa

```kotlin
@Composable
fun ScoreCounter(initialScore: Int) {
    var score by remember { mutableStateOf(initialScore) }

    Row {
        Text("$score")
        Button(onClick = { score++ }) { Text("+1") }
    }
}

// Uso en la pantalla:
ScoreCounter(initialScore = state.currentScore)
```

La trampa: parece funcionar — el contador incrementa visualmente al tocar el botón. Pero `score` es un `remember` *local* al composable, inicializado una sola vez con `initialScore`. Si el `ViewModel` actualiza `state.currentScore` por otra vía (por ejemplo, otro jugador sincronizado vía Firebase, o una corrección de puntaje desde otra pantalla), `ScoreCounter` **no se entera** — sigue mostrando su copia local, desincronizada del verdadero estado. El bug es silencioso: compila, corre, y visualmente "funciona" en el caso feliz de un solo usuario tocando el botón una sola vez, pero rompe la garantía de fuente única de verdad en cuanto hay más de un camino que puede cambiar el score. La solución es hoistear: `ScoreCounter(score: Int, onIncrement: () -> Unit)`, dejando que el `ViewModel` sea la única fuente de verdad, tal como exige MVI.

## 6. Conexión con arquitectura real

State hoisting es, en Timbax, la bisagra entre `presentation` y `ui` documentada implícitamente desde `06_presentation_mvi/contract_state_event_effect.md`: el `PlayersState` que expone el `ViewModel` como `StateFlow` es el estado "hoisted al máximo" — la fuente de verdad única de toda la pantalla. Cada composable de UI (`PlayerCard`, `AddPlayerForm` de `material3_componentes_comunes.md`) es stateless y recibe fragmentos de ese `State` más callbacks que terminan emitiendo un `Event` hacia el `ViewModel`. La cadena completa —`ViewModel` (dueño del estado) → `Screen` (lee `State`, hoistea hacia los hijos) → composables hijos (stateless, solo `value`/`onValueChange`)— es la aplicación literal de Clean Architecture dentro de la capa de UI: cada nivel conoce solo lo que necesita, y la fuente de verdad nunca se duplica.