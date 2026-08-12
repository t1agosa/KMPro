# material3_componentes_comunes.md

## 1. Qué es

Material3 (M3) es la implementación de Compose de la especificación Material Design 3 de Google: un set de componentes de UI listos para usar (`Button`, `TextField`, `Card`, entre otros) que ya vienen con estilo, estados (enabled/disabled, focused, error) y comportamiento accesible por default. No son layouts (no organizan espacio) — son piezas de contenido con las que se arma la interfaz dentro de esos layouts.

Este archivo cubre los tres que más vas a usar día a día: `Button` (y sus variantes), `TextField`, y `Card`.

## 2. El problema que resuelve

Sin un sistema de diseño, cada botón, cada input de texto y cada card tendrían que construirse desde cero combinando `Box`/`Row`/`Modifier` — reimplementando manualmente estados de foco, ripple de click, elevación, colores por tema, contraste accesible, etc. Material3 encapsula todo eso en componentes con una API estable y consistente, para que la decisión de diseño no sea "cómo construyo un botón" sino "qué variante de botón le corresponde semánticamente a esta acción".

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun AddPlayerForm(
    name: String,
    onNameChange: (String) -> Unit,
    onSaveClick: () -> Unit,
    isSaving: Boolean,
    modifier: Modifier = Modifier
) {
    Card(modifier = modifier.fillMaxWidth()) {
        Column(
            modifier = Modifier.padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp)
        ) {
            TextField(
                value = name,               // state hoisting: el valor vive afuera
                onValueChange = onNameChange, // el TextField no guarda su propio estado
                label = { Text("Nombre del jugador") },
                singleLine = true
            )

            Button(
                onClick = onSaveClick,
                enabled = name.isNotBlank() && !isSaving, // regla de UI, no de negocio
                modifier = Modifier.align(Alignment.End)
            ) {
                Text(if (isSaving) "Guardando..." else "Guardar")
            }
        }
    }
}
```

`TextField` sigue el mismo patrón de state hoisting que ya vimos: no guarda `name` internamente, recibe `value` y `onValueChange` desde afuera — así el `ViewModel` (a través del `State`) es la única fuente de verdad, incluso para lo que el usuario está tipeando.

## 4. Matriz de criterio

**Variantes de `Button` (`Button`, `OutlinedButton`, `TextButton`, `FilledTonalButton`)**
- `Button` (filled, color sólido): usar para la acción principal de la pantalla — solo debería haber una por pantalla o sección (ej: "Guardar", "Confirmar partida").
- `OutlinedButton`: usar para una acción secundaria, presente junto a la principal (ej: "Cancelar" al lado de "Guardar").
- `TextButton`: usar para acciones de bajo énfasis visual, como en un diálogo ("Ahora no") o dentro de una `TopAppBar`.
- NO usar cuando: uses varios `Button` (filled) en la misma pantalla — eso rompe la jerarquía visual, porque todas las acciones compiten por la misma atención.
- Trade-off: elegir la variante correcta no es estético, es semántico — comunica al usuario cuál es la acción esperada sin que tenga que leer el texto.

**`TextField` (`TextField` vs `OutlinedTextField`)**
- Usar `TextField` (filled) o `OutlinedTextField` según el resto del sistema de diseño de la app — ambos cumplen la misma función, difieren en estilo visual (fondo relleno vs borde). Elegir uno y mantenerlo consistente en toda la app.
- NO usar cuando: necesitás manejar estado propio del texto dentro del componente — siempre hoisteado, igual que cualquier otro estado de UI (ver `remember_vs_remembersaveable.md` para el caso en que sí conviene un estado puramente local y efímero).
- Trade-off: requiere que quien lo usa mantenga el `value` en algún `State` — más verboso que un input "mágico" autocontenido, pero es justamente lo que lo hace testeable y consistente con MVI.

**`Card`**
- Usar cuando: necesitás agrupar contenido relacionado con una superficie visualmente diferenciada (elevación, borde redondeado) — ítems de lista, formularios, secciones destacadas.
- NO usar cuando: es solo un contenedor de layout sin necesidad de diferenciación visual — ahí un `Column`/`Box` simple alcanza y evita elevación/sombra innecesaria.
- Trade-off: trae estilo gratis, pero también trae opinión de diseño (elevación, forma) — si tu diseño necesita algo muy distinto al default de M3, puede terminar peleando contra el componente en vez de ayudarte.

## 5. Caso trampa

```kotlin
Button(
    onClick = { onEvent(PlayersEvent.OnSaveClicked) },
) {
    Text("Guardar")
}
```

La trampa: este botón *siempre* está habilitado, incluso si `name` está vacío o si ya hay un guardado en curso (`isSaving`). No hay ningún error de compilación ni de runtime — simplemente el usuario puede tocar "Guardar" con un formulario inválido, y el `Event` se dispara igual hacia el `ViewModel`. La regla de habilitación (`enabled = name.isNotBlank() && !isSaving`) es una decisión de **UI**, calculada a partir del `State`, y hay que declararla explícitamente en el composable — Compose no la infiere ni la exige. Confundir esto con "eso lo valida el `UseCase`" es un error de capa: el `UseCase` debe validar igual (nunca confiar solo en la UI), pero deshabilitar el botón es lo que le da al usuario feedback inmediato sin necesidad de ida y vuelta al `ViewModel`.

## 6. Conexión con arquitectura real

En Timbax, `enabled = name.isNotBlank() && !isSaving` es un cálculo derivado directamente de `PlayersState` — nunca una decisión que vive suelta en el composable sin relación al `State`. `isSaving` en particular normalmente es un campo del mismo `State` que ya se usa para mostrar loading en otras partes de la pantalla (mismo patrón documentado en `02_domain/result_pattern.md`, donde se aclaró que el `isLoading` vive en `presentation`, no en el `Result` del dominio). El componente M3 (`Button`) es solo el vehículo visual de una regla que ya estaba resuelta en el `State` que bajó del `ViewModel`.