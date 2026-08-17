# Navigation 3 (Nav3)

## 1. Qué es

Navigation 3 (Nav3) es la reescritura de Google de la librería de navegación para Compose, con una arquitectura fundamentalmente distinta a `Navigation Compose` 2.x (la documentada en `navegacion.md`). El cambio central: en 2.x, el `NavController` esconde el back stack adentro suyo y expone rutas como `String` (`"player_detail/${id}"`); en Nav3, **el back stack es tuyo** — una lista observable de Compose (`SnapshotStateList<NavKey>`) que vos creás y mutás directamente (`backStack.add(...)`, `backStack.removeLastOrNull()`), y cada destino es un tipo Kotlin real (`data class`/`data object` que implementa `NavKey`), no un string. `NavDisplay` reemplaza a `NavHost`: observa esa lista y decide qué renderizar vía un `entryProvider` que mapea cada tipo de destino a su `NavEntry`.

La otra pieza importante para este repo: desde Compose Multiplatform 1.10 (JetBrains, 2026), Navigation 3 tiene soporte multiplataforma oficial — Android, iOS, Desktop y Web — vía artefactos espejo publicados por JetBrains (`org.jetbrains.androidx.navigation3:navigation3-ui`). Es la primera vez que la navegación "oficial de Google" deja de ser Android-only, algo que hasta ahora era la razón principal por la que un proyecto KMP-first elegía Voyager o Decompose por sobre Navigation Compose (ver `cuando_elegir_cada_stack_option.md`).

## 2. El problema que resuelve

`Navigation Compose` 2.x seguía usando por debajo mecanismos pensados originalmente para Fragments/XML — rutas como `String` que se resuelven en runtime (con errores de tipeo invisibles hasta que la app crashea), un `NavController` que abstrae el back stack a tal punto que testearlo o razonar sobre "qué hay en la pila ahora mismo" es indirecto, y una dificultad real para armar layouts adaptativos (mostrar lista+detalle simultáneamente en tablet, por ejemplo) porque el modelo asume una sola pantalla visible a la vez. Nav3 resuelve los tres problemas de una: tipos en vez de strings (errores de destino inexistente los atrapa el compilador, no el usuario en producción), el back stack como estado explícito y testeable, y una capa de renderizado (`NavDisplay` + `Scenes`) desacoplada del back stack, que permite que un mismo estado de navegación se muestre distinto según el tamaño de pantalla.

## 3. Ejemplo mínimo comentado

```kotlin
// Destinos: tipos reales, no strings — serializables para sobrevivir process death
@Serializable
sealed interface Screen : NavKey {
    @Serializable data object PlayersList : Screen
    @Serializable data class PlayerDetail(val playerId: String) : Screen
}
```

```kotlin
@Composable
fun TimbaxNavigation() {
    // el back stack es tu propio estado — una lista observable que vos controlás
    val backStack = rememberNavBackStack(Screen.PlayersList)

    NavDisplay(
        backStack = backStack,
        onBack = { backStack.removeLastOrNull() },
        entryProvider = entryProvider {
            entry<Screen.PlayersList> {
                PlayersScreen(onPlayerClick = { id -> backStack.add(Screen.PlayerDetail(id)) })
            }
            entry<Screen.PlayerDetail> { screen ->
                PlayerDetailScreen(playerId = screen.playerId)
            }
        }
    )
}
```

`rememberNavBackStack(Screen.PlayersList)` arranca la pila con un solo destino. `entry<Screen.PlayerDetail> { screen -> ... }` recibe el destino tipado directamente (`screen.playerId` con autocompletado y chequeo de tipos), sin ningún `navController.getArgument("playerId")` a ciegas como en 2.x.

## 4. Matriz de criterio

**Migrar a Nav3 vs. quedarse en `Navigation Compose` 2.x:**
- Migrar cuando: arranca un proyecto nuevo, o el proyecto actual sufre activamente los problemas de rutas string (errores en runtime, dificultad para testear el back stack) y puede absorber una librería todavía en evolución activa (los artefactos de Nav3 siguen moviéndose de versión con cierta frecuencia).
- Quedarse en 2.x cuando: el proyecto ya tiene una integración de navegación estable y funcionando, sin dolor real, y no hay urgencia por layouts adaptativos multi-panel — migrar "porque es lo nuevo" sin un problema concreto que resolver no se justifica.
- Trade-off: Nav3 da type-safety y testeabilidad real del back stack, a cambio de ser una librería más joven, con más superficie de API en movimiento que la ya asentada `Navigation Compose` 2.x.

**Nav3 vs. Voyager/Decompose, ahora que Nav3 tiene soporte KMP oficial:**
- Usar Nav3 cuando: el proyecto quiere quedarse lo más cerca posible del ecosistema oficial de Google/JetBrains, y puede tolerar una librería que a mediados de 2026 todavía está estabilizando su versión final — el criterio "KMP-first → Voyager/Decompose" de `cuando_elegir_cada_stack_option.md` ya no es tan tajante como cuando se escribió ese archivo, porque la razón principal detrás de esa regla (Navigation Compose no soportaba iOS/Desktop/Web de forma oficial) dejó de ser cierta.
- Seguir con Voyager/Decompose cuando: el proyecto ya está construido sobre esa base y funciona bien, o se prioriza una API más madura/estable hoy por sobre estar en la vanguardia de lo que publica Google/JetBrains.
- Trade-off: este es exactamente el tipo de decisión que corresponde revisar en `14_criterio_y_decisiones` cuando ese package se actualice — no es una corrección al archivo de navegación existente, es una opción nueva que antes no estaba disponible.

**Back stack como `SnapshotStateList` propio vs. `NavController` encapsulado (2.x):**
- Usar Nav3 cuando: te importa poder testear "después de esta secuencia de acciones, ¿qué hay en la pila?" sin mockear un `NavController` — el back stack de Nav3 es una lista Kotlin normal, se puede inspeccionar y aserciones directas sobre su contenido en un test.
- Mantener el patrón vía `Effect` (documentado en `navegacion.md`) sin importar cuál de las dos librerías uses — que el back stack ahora sea "tu propia lista" no cambia quién debería decidir mutarla (ver Caso trampa).

**Serialización de destinos: reflection (Android) vs. polymorphic serialization explícita (KMP):**
- Usar reflection-based serialization (el comportamiento por default) cuando: el target es solo Android — la JVM permite que Nav3 resuelva la serialización de los destinos automáticamente.
- Declarar `polymorphic serialization` explícita para los tipos `NavKey` cuando: el proyecto tiene targets no-JVM (iOS, Web) — esas plataformas no soportan reflection, así que sin este paso extra el back stack no puede serializarse para sobrevivir process death fuera de Android.
- Trade-off: un paso de configuración extra específico de KMP, a cambio de que el mismo código de navegación tipada funcione de verdad en las tres plataformas, no solo en Android.

## 5. Caso trampa

Mutar el back stack directamente desde un composable hoja, razonando que "ahora es solo una lista, así que ya no hay problema en tocarla desde cualquier lado":

```kotlin
// ❌ trampa: el backStack "ahora es solo una lista", entonces parece razonable
// mutarla directo desde el composable — total, ya no hay NavController escondiendo nada
@Composable
fun PlayerCard(player: Player, backStack: SnapshotStateList<Screen>) {
    Card(
        modifier = Modifier.clickable {
            backStack.add(Screen.PlayerDetail(player.id)) // navegación directa, otra vez
        }
    ) {
        Text(player.name)
    }
}
```

El cambio de filosofía de Nav3 ("el back stack es tu propio estado") es una mejora de **control e implementación** — no un permiso para saltarse la arquitectura MVI. Mutar el `backStack` directo desde un composable hoja reproduce exactamente el mismo problema ya documentado en `navegacion.md` para `Navigation Compose` 2.x: la decisión de "a dónde ir" queda atrapada en la capa de UI, imposible de testear con un test de `ViewModel` puro, y `PlayerCard` deja de ser reusable fuera de un contexto con acceso directo a esa lista. Que el back stack ahora sea una `SnapshotStateList` normal es justamente lo que hace *más fácil* testear al componente que sí debería tener acceso a ella (la `Screen` de nivel más alto que colecta los `Effect` del `ViewModel`) — no una invitación a que cualquier composable hijo la mute. El loop correcto sigue siendo el mismo: `Event` → el `ViewModel` decide → `Effect` → el composable que efectivamente conoce el `backStack` reacciona mutándolo.

## 6. Conexión con Timbax

Timbax hoy no usa Navigation 3 — la elección de navegación se resuelve con el criterio ya documentado en `navegacion.md` y `cuando_elegir_cada_stack_option.md` (Navigation Compose vs. Voyager/Decompose, según si el proyecto es Android-first o KMP-first). Este archivo existe para dejar registrado que esa matriz cambió de insumos: con Nav3 alcanzando soporte KMP oficial vía JetBrains desde Compose Multiplatform 1.10, la razón histórica más fuerte para preferir Voyager/Decompose en un proyecto KMP-first (que Navigation Compose no corría fuera de Android) ya no aplica igual. No implica que Timbax deba migrar — implica que la próxima vez que `14_criterio_y_decisiones` se revise, esta es una opción real a sopesar, con el mismo patrón de Effect que el resto de la arquitectura de Timbax ya usa para navegación, sin importar qué librería termine implementándolo por debajo.