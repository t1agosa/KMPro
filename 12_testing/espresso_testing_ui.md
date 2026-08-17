# Espresso (UI Testing Instrumentado)

## 1. Qué es

Espresso es el framework oficial de Google para tests de UI **instrumentados** — tests que corren dentro de un dispositivo o emulador Android real, interactuando con la jerarquía de `View` de verdad, no simulada. Su API se organiza en tres piezas que se combinan siempre en el mismo orden: `ViewMatcher` (encontrar la vista — `withId()`, `withText()`), `ViewAction` (hacer algo — `click()`, `typeText()`) y `ViewAssertion` (verificar el resultado — `matches(isDisplayed())`), todo orquestado por el punto de entrada `onView(...)`. Nació para apps basadas en `View`/XML — con Compose, ese rol pasó en gran parte a la **Compose Testing Library** (`ComposeTestRule`), que opera sobre el árbol de semántica (ya documentado en `accesibilidad_a11y.md`) en vez de la jerarquía de `View`, con una filosofía casi idéntica pero una implementación completamente distinta y no intercambiable.

## 2. El problema que resuelve

Los tests unitarios de `UseCase`/`ViewModel` (documentados en `estrategia_y_prioridades.md` y `fakes_vs_mocks_turbine.md`) corren en la JVM, sin tocar un dispositivo real — verifican lógica, no que la UI efectivamente se vea o reaccione como se espera. Un test instrumentado como Espresso resuelve esa capa faltante: corre con el framework de Android real, verificando que un click de verdad dispare la acción esperada, que un texto aparezca en pantalla, que una transición ocurra. El problema específico que Espresso ataca dentro de esa categoría es la **sincronización**: sin ella, un test podría intentar aserción sobre una vista antes de que una animación o llamada asincrónica termine, dando resultados intermitentes (a veces pasa, a veces no) — Espresso espera automáticamente a que la cola de mensajes del hilo principal esté idle antes de ejecutar el siguiente paso, eliminando la mayoría de esos falsos negativos sin que el desarrollador tenga que agregar esperas manuales.

## 3. Ejemplo mínimo comentado

```kotlin
// Test instrumentado clásico de Espresso, sobre una View XML (no Compose)
@Test
fun clickingSaveButton_showsConfirmationMessage() {
    onView(withId(R.id.saveButton))
        .perform(click())

    onView(withId(R.id.confirmationText))
        .check(matches(isDisplayed()))
}
```

```kotlin
// Compose (lo que usa Timbax) — misma idea, pero sobre el árbol de semántica, no la View hierarchy
@get:Rule
val composeTestRule = createAndroidComposeRule<MainActivity>()

@Test
fun clickingSaveButton_showsConfirmationMessage() {
    composeTestRule.onNodeWithText("Guardar").performClick()
    composeTestRule.onNodeWithText("Puntaje guardado").assertIsDisplayed()
}
```

`onView`/`perform`/`check` (Espresso) y `onNode`/`performClick`/`assertIsDisplayed` (Compose Testing) siguen la misma filosofía de tres pasos — encontrar, actuar, verificar — pero son APIs completamente separadas que no se entienden entre sí por default: Espresso no puede encontrar nodos de Compose, y Compose Testing no puede encontrar `View` clásicas. `createAndroidComposeRule` es el puente que permite combinar ambas en la misma clase de test cuando hace falta (ver Matriz de criterio).

## 4. Matriz de criterio

**Espresso vs. Compose Testing Library:**
- Usar Compose Testing (`ComposeTestRule`) cuando: la pantalla es 100% Compose — es el caso general en un proyecto como Timbax. Corre más rápido que Espresso para tests aislados (no siempre necesita un emulador completo) y es menos propenso a flakiness porque su sincronización conoce directamente el ciclo de recomposición.
- Usar Espresso cuando: el proyecto tiene `View` clásicas (XML) en algún punto de la app — legado, o interop de una librería de terceros que expone una `View` (un `WebView`, un SDK de mapas, un banner de ads) embebida dentro del árbol de Compose.
- Trade-off: para un proyecto Compose-only, mantener Espresso sin necesidad real es carga extra de dependencias y conceptos sin beneficio — pero es exactamente la palabra clave que un reclutador de Android nativo espera ver, incluso si el uso real en el día a día es marginal.

**`createComposeRule()` vs. `createAndroidComposeRule<Activity>()`:**
- Usar `createComposeRule()` cuando: el test es de un composable aislado, sin necesidad de una `Activity` real — más liviano, arranca una `ComponentActivity` vacía por detrás.
- Usar `createAndroidComposeRule<MainActivity>()` cuando: el test necesita el contexto real de la `Activity` de la app (para interop con `View`, para acceder a algo que depende del `Application`/DI real) — es el que habilita combinar Espresso y Compose Testing en el mismo test.
- Trade-off: `createAndroidComposeRule` es más pesado (levanta la `Activity` real de la app) pero es indispensable en cualquier escenario de interop.

**Combinar Espresso + Compose Testing (interop) vs. usar solo uno de los dos:**
- Combinar ambos cuando: la pantalla mezcla `View` clásicas y Compose en el mismo árbol — se matchean las `View` con `onView` (Espresso) y los elementos Compose con `composeTestRule.onNode` (Compose Testing), sin pasos especiales adicionales más allá de usar `createAndroidComposeRule`.
- Usar solo uno cuando: la pantalla es enteramente de un solo tipo — mezclar frameworks sin necesidad real solo agrega complejidad al test sin beneficio.

**Tests instrumentados (Espresso/Compose Testing) vs. tests unitarios de `ViewModel` con Turbine:**
- Priorizar tests unitarios con Turbine (ya documentado en `fakes_vs_mocks_turbine.md`) cuando: lo que se quiere verificar es lógica de estado — corren en la JVM, sin emulador, en milisegundos, y son la prioridad real según `estrategia_y_prioridades.md`.
- Reservar tests instrumentados para: los flujos críticos de punta a punta que sí necesitan verificarse contra el framework real de Android (un flujo de login completo, una compra) — son más lentos, más caros de mantener, y por eso se usan con moderación, no como reemplazo de los tests unitarios.

## 5. Caso trampa

Asumir que Espresso sincroniza automáticamente con **cualquier** trabajo asincrónico, incluidas coroutines lanzadas por la app:

```kotlin
// ❌ trampa: el ViewModel lanza una coroutine en background al guardar el puntaje,
// pero el test no le avisa nada a Espresso sobre ese trabajo
@Test
fun savingScore_showsConfirmation() {
    onView(withId(R.id.saveButton)).perform(click())
    // acá el ViewModel dispara viewModelScope.launch { repository.savePlayer(...) }

    onView(withId(R.id.confirmationText)).check(matches(isDisplayed())) // 🚩 flaky
}
```

Espresso solo sincroniza automáticamente con trabajo que pasa por la cola de mensajes del hilo principal de Android (dibujo de `View`, el viejo `AsyncTask`). No tiene ninguna visibilidad sobre coroutines, streams de `Flow`, llamadas de red, o cualquier trabajo en threads/executors custom — desde su perspectiva, esas operaciones son invisibles. El test de arriba puede pasar la mayoría de las veces (si la coroutine termina rápido) y fallar intermitentemente cuando no (si el dispositivo está lento, si hay contención de CPU en CI) — el síntoma clásico de un test flaky, difícil de reproducir en la propia máquina y frustrante en CI. La corrección es un `IdlingResource` (típicamente `CountingIdlingResource`): se incrementa un contador cuando arranca el trabajo asincrónico relevante y se decrementa cuando termina, y Espresso espera mientras ese contador esté por encima de cero antes de seguir con el siguiente paso del test.

La señal de alarma: cualquier test de Espresso que involucre una acción que dispara una `suspend fun`/coroutine (guardar en DB, llamar a la red) sin ningún `IdlingResource` registrado es candidato directo a flakiness — funciona en la demo, falla al azar en CI.

## 6. Conexión con Timbax

Timbax es 100% Compose, sin `View` XML — así que en el día a día, sus tests de UI (si se escriben, siguiendo la prioridad baja que ya documenta `estrategia_y_prioridades.md`: domain y presentation primero, UI casi nunca) usarían `ComposeTestRule` directamente, no Espresso puro. Espresso entra al mapa de Timbax en dos escenarios concretos: como la palabra clave que un reclutador de Android nativo espera ver en el CV (el motivo de fondo por el que este archivo existe en el repo), y como herramienta real si algún día Timbax embebiera una `View` nativa dentro de su árbol de Compose — por ejemplo, un `AdView` de un SDK de terceros, si el proyecto decidiera monetizar con ads (una decisión que hoy no está tomada, y que si se toma, debería documentarse en `14_criterio_y_decisiones` igual que las de los módulos 15-17). En ese escenario puntual, el patrón de `createAndroidComposeRule` combinando `onView` y `composeTestRule.onNode` en el mismo test es exactamente lo que haría falta.