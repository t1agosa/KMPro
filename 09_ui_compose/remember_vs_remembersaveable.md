# remember_vs_remembersaveable.md

## 1. Qué es

`remember` y `rememberSaveable` son APIs para guardar un valor en memoria dentro de un composable, sobreviviendo a recomposiciones sucesivas. La diferencia entre ambos está en **qué eventos del ciclo de vida sobreviven**:

- `remember`: sobrevive a recomposiciones, pero se pierde si el composable sale de la composición (navegás a otra pantalla y volvés) o si hay un cambio de configuración (rotación de pantalla, cambio de idioma del sistema).
- `rememberSaveable`: además de sobrevivir recomposiciones, sobrevive cambios de configuración, porque serializa el valor a un mecanismo de persistencia liviano (el `Bundle` en Android, o el equivalente por plataforma en KMP) y lo restaura automáticamente.

## 2. El problema que resuelve

Sin `remember`, cualquier variable local declarada dentro del cuerpo de un composable (`var expanded = false`) se reinicializaría en **cada** recomposición, porque el cuerpo de la función vuelve a ejecutarse de punta a punta — perdiendo cualquier estado de UI efímero cada vez que Compose decide recomponer, aunque sea por un motivo completamente ajeno a esa variable.

`remember` resuelve eso, pero solo dentro de los límites de la composición actual: no protege contra un cambio de configuración, que en Android históricamente recreaba la Activity entera. `rememberSaveable` resuelve ese segundo problema, agregando persistencia real (aunque liviana y temporal) para que el usuario no pierda estado de UI visible al rotar la pantalla — algo que rompería la expectativa básica de continuidad que tiene cualquier usuario.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayerCard(player: Player) {
    // sobrevive recomposiciones, pero se resetea si rotás la pantalla
    var isExpanded by remember { mutableStateOf(false) }

    Card(
        modifier = Modifier.clickable { isExpanded = !isExpanded }
    ) {
        Text(player.name)
        if (isExpanded) {
            Text("Score: ${player.score}")
        }
    }
}

@Composable
fun AddPlayerForm() {
    // sobrevive recomposiciones Y rotación de pantalla
    var name by rememberSaveable { mutableStateOf("") }

    TextField(value = name, onValueChange = { name = it })
}
```

Si el usuario expande `PlayerCard` (`isExpanded = true`) y rota el dispositivo, la card vuelve a mostrarse colapsada — comportamiento aceptable para un detalle visual menor. Pero si el usuario escribió medio nombre en `AddPlayerForm` con solo `remember` y rota la pantalla, perdería lo tipeado — por eso ahí corresponde `rememberSaveable`.

## 4. Matriz de criterio

**`remember`**
- Usar cuando: el estado es visual, efímero, y perderlo en una rotación no afecta la experiencia del usuario de forma notoria (si una card está expandida, si un tooltip está visible, un `Animatable` en curso).
- NO usar cuando: el usuario esperaría razonablemente que ese valor sobreviva a rotar el dispositivo (texto tipeado, un checkbox marcado en un formulario largo).

**`rememberSaveable`**
- Usar cuando: el estado es de UI pero el usuario lo percibiría como "perdido" si desaparece al rotar — inputs de formulario, posición de scroll relevante, selección en un picker.
- NO usar cuando: el tipo no es serializable de forma directa (objetos complejos, sin soporte nativo) — ahí hace falta un `Saver` custom, o directamente reconsiderar si ese estado no debería vivir en el `ViewModel` en primer lugar.
- Trade-off: agrega el costo (mínimo, pero real) de serialización en cada configuración change — normalmente irrelevante para valores primitivos simples.

**`remember`/`rememberSaveable` vs `State` del `ViewModel`**
- Usar `remember`(`Saveable`) cuando: el estado es puramente de presentación visual, sin relevancia de negocio ni necesidad de sobrevivir más allá de esa pantalla.
- Usar el `State` del `ViewModel` cuando: el dato tiene relevancia de negocio, necesita persistir más allá de un cambio de configuración (por ejemplo, sobrevivir a que el usuario navegue a otra pantalla y vuelva), o debe ser accedido/testeado fuera de la UI.
- Trade-off: el `ViewModel` ya sobrevive cambios de configuración por su propio ciclo de vida (ver la pregunta típica más abajo) — así que para datos de negocio, ni siquiera hace falta `rememberSaveable`; el problema que resuelve esa API ya está resuelto de otra forma en ese nivel.

## 5. Caso trampa

```kotlin
@Composable
fun AddPlayerScreen(viewModel: AddPlayerViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    // "para no tener que tocar el ViewModel", el dev usa remember local
    var localName by remember { mutableStateOf(state.name) }

    TextField(
        value = localName,
        onValueChange = { localName = it } // nunca llega al ViewModel
    )

    Button(onClick = { viewModel.onEvent(AddPlayerEvent.OnSaveClicked(localName)) }) {
        Text("Guardar")
    }
}
```

La trampa: esto "funciona" en el caso feliz — el usuario tipea, toca Guardar, y el nombre se guarda. Pero se rompió el principio de **state hoisting** (`composables_y_state_hoisting.md`) sin que sea obvio a simple vista: `localName` es ahora una segunda fuente de verdad, desincronizada de `state.name`. Si algo más en la pantalla necesita reaccionar al nombre mientras se tipea (validación en vivo, un contador de caracteres calculado en el `ViewModel`, otro composable que muestre un preview), no va a funcionar, porque el `ViewModel` no se entera de cada tecla — solo se entera al final, cuando se toca "Guardar". Además, si la pantalla rota, `remember` reinicializa `localName` con el `state.name` de ese momento (perdiendo lo tipeado sin guardar), mientras que si el flujo fuera `onNameChange` disparando un `Event` hacia el `ViewModel` en cada tecla, el `State` del `ViewModel` ya sobrevive la rotación sin necesitar ningún `remember` en absoluto.

## 6. Conexión con arquitectura real

En Timbax, la pregunta "¿por qué el `State` del `ViewModel` no necesita `rememberSaveable`?" tiene una respuesta arquitectónica concreta: el `ViewModel` sobrevive cambios de configuración por su propio scope de ciclo de vida (atado a la navegación, no a la Activity/Composable en sí — documentado también en `07_coroutines/fundamentos_suspend_scope.md` respecto a `viewModelScope`), así que ese problema ya está resuelto en una capa más arriba. `remember`/`rememberSaveable` en Timbax se reservan exclusivamente para estado que genuinamente nace y muere en la UI (si un diálogo de confirmación está abierto, si una card de historial está expandida) — nunca como atajo para evitar declarar un `Event` nuevo hacia el `ViewModel`.