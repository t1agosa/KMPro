# animaciones_basicas.md

## 1. Qué es

Compose ofrece un set de APIs de animación de alto nivel, pensadas para cubrir la gran mayoría de casos comunes sin necesitar manejar interpolación manual de valores: `animate*AsState()` (una familia de funciones: `animateFloatAsState`, `animateColorAsState`, `animateDpAsState`, etc.) para animar un solo valor entre dos estados; `AnimatedVisibility` para animar la entrada/salida de un composable completo; y `animateContentSize()`, un `Modifier` que anima automáticamente el cambio de tamaño de un composable cuando su contenido cambia.

Todas comparten la misma idea de fondo: en vez de escribir la animación como una secuencia de pasos imperativos, se declara el **valor objetivo** (`targetValue`) y Compose se encarga de interpolar suavemente desde el valor actual hasta ese objetivo cada vez que cambia.

## 2. El problema que resuelve

Animar manualmente en un sistema imperativo implica gestionar un timer, calcular manualmente los valores intermedios frame a frame, y sincronizar eso con el render — mucho código repetitivo para casos que en la práctica son muy comunes (un color que cambia suavemente, un elemento que aparece con fade, una card que se expande).

Las animate-APIs de Compose resuelven esto reduciendo la animación a una declaración: "cuando este valor cambia, quiero que la transición sea animada, no instantánea". Compose se encarga de todo el trabajo de interpolación y de disparar recomposición en cada frame intermedio de la animación.

## 3. Ejemplo mínimo comentado

```kotlin
@Composable
fun PlayerCard(player: Player, isSelected: Boolean, onClick: () -> Unit) {
    // 1. animateColorAsState: interpola el color cada vez que isSelected cambia
    val backgroundColor by animateColorAsState(
        targetValue = if (isSelected) MaterialTheme.colorScheme.primaryContainer
                      else MaterialTheme.colorScheme.surface,
        label = "cardBackground"
    )

    Card(
        modifier = Modifier
            .clickable(onClick = onClick)
            .animateContentSize(), // 3. anima el cambio de tamaño cuando isSelected expande el contenido
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        Column(modifier = Modifier.padding(12.dp)) {
            Text(player.name)

            // 2. AnimatedVisibility: anima la entrada/salida del bloque completo
            AnimatedVisibility(visible = isSelected) {
                Text("Score: ${player.score}", style = MaterialTheme.typography.bodySmall)
            }
        }
    }
}
```

Cuando `isSelected` cambia (viene del `State` del `ViewModel`, no de un `remember` local — ver `composables_y_state_hoisting.md`), tres cosas animan a la vez: el color de fondo interpola suavemente (`animateColorAsState`), el detalle del score aparece/desaparece con una transición (`AnimatedVisibility`), y el tamaño de la `Card` se ajusta de forma animada al nuevo contenido (`animateContentSize()`) — sin que el desarrollador haya escrito ningún cálculo de interpolación a mano.

## 4. Matriz de criterio

**`animate*AsState()`**
- Usar cuando: necesitás animar un único valor simple (color, tamaño, alpha, offset) entre dos estados conocidos — es la opción de menor esfuerzo para el caso más común.
- NO usar cuando: necesitás coordinar varias animaciones relacionadas entre sí con timing preciso (secuencias, animaciones que dependen unas de otras) — ahí corresponde `Animatable` o `updateTransition`, APIs de nivel más bajo que exceden el alcance de este archivo introductorio.
- Trade-off: mínimo código a cambio de control limitado — no permite, por ejemplo, encadenar múltiples animaciones con distinto timing sin recurrir a APIs más avanzadas.

**`AnimatedVisibility`**
- Usar cuando: un composable completo debe aparecer/desaparecer de forma animada en respuesta a un `Boolean` — reemplaza un `if (visible) { Content() }` abrupto por una transición real (fade, slide, expand, por default o combinables).
- NO usar cuando: el composable simplemente no debería estar en el árbol en absoluto (por ejemplo, contenido que depende de una feature flag deshabilitada permanentemente) — ahí un `if` normal alcanza y es más simple.
- Trade-off: agrega una capa de composición extra (mide entrada y salida) — irrelevante en la práctica salvo en árboles extremadamente grandes con animaciones simultáneas.

**`Modifier.animateContentSize()`**
- Usar cuando: el contenido interno de un composable puede cambiar de tamaño (texto que se expande, una lista que crece) y preferís una transición suave del contenedor en vez de un salto brusco de tamaño.
- NO usar cuando: el cambio de tamaño es tan frecuente o tan grande que la animación constante termina sintiéndose como ruido visual en vez de ayuda (ej: un contenido que cambia de tamaño en cada frame de scroll) — en esos casos, sin animación es más legible.

**Duración/easing por default vs custom (`tween`, `spring`)**
- Usar los defaults cuando: no hay una razón de diseño específica para desviarse — los valores por default de Compose ya están pensados para sentirse naturales en la mayoría de los casos.
- Customizar (`animationSpec = tween(300, easing = FastOutSlowInEasing)` o `spring(...)`) cuando: el sistema de diseño de la app define timings/curvas específicas, o cuando el default se siente demasiado lento/rápido para ese caso puntual.

## 5. Caso trampa

```kotlin
@Composable
fun ScoreCounter(score: Int) {
    // se recalcula en CADA recomposición, no solo cuando score cambia
    val animatedColor = if (score > 100) Color.Green else Color.Gray

    Text("$score", color = animatedColor)
}
```

La trampa: el nombre de la variable (`animatedColor`) sugiere que hay una animación, pero no la hay — es un `if` simple evaluado en cada recomposición, así que el color **salta** instantáneamente de gris a verde apenas `score` cruza 100, sin ninguna transición. Es un error fácil de cometer porque *funciona* visualmente (el color correcto termina apareciendo), pero no cumple la intención original de que el cambio se sienta suave. La corrección es reemplazar el `if` directo por `animateColorAsState(targetValue = if (score > 100) Color.Green else Color.Gray)` — la diferencia de una sola función envolviendo la expresión es lo que activa la interpolación real de Compose en vez de un salto abrupto.

## 6. Conexión con arquitectura real

En Timbax, `isSelected` o `score > 100` (como en el ejemplo de `ScoreCounter`) son valores que siempre se derivan del `State` que expone el `ViewModel` — nunca de un estado local inventado solo para la animación. Las animate-APIs de este archivo son la capa final, puramente visual, sobre una cadena que ya documentamos completa: `ViewModel` decide el dato (`PlayersState`) → `Screen` lo colecta y desestructura (`collect_stateflow_en_compose.md`) → el composable hoja recibe el valor puntual (`composables_y_state_hoisting.md`) → y recién ahí, si corresponde, se envuelve ese valor en una animate-API para que el cambio se sienta suave en vez de abrupto. La animación nunca decide *qué* mostrar — solo *cómo* transiciona algo que el `State` ya decidió.