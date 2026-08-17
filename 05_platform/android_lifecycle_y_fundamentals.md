# Android Lifecycle y Fundamentals

## 1. Qué es

El lifecycle de Android es el conjunto de estados por los que pasa una `Activity` durante su existencia, gestionados por el sistema operativo — no por el desarrollador. Los 6 callbacks core son `onCreate()`, `onStart()`, `onResume()`, `onPause()`, `onStop()` y `onDestroy()` (más `onRestart()`, una transición especial de Stopped → Started que no siempre se invoca). En una app Compose/Compose Multiplatform como Timbax, la `Activity` deja de ser "la pantalla" — pasa a ser un contenedor único (`ComponentActivity`) que monta, una sola vez, el árbol entero de Compose vía `setContent { }`. Es contenido que solo existe en `androidMain`: no hay `expect/actual` que bridgear acá, porque iOS no tiene el concepto de `Activity` (usa `UIViewController`) y Compose Multiplatform ya abstrae esa diferencia por vos.

Junto con el lifecycle va `Context`: la interfaz que da acceso a recursos del sistema (strings, layouts, servicios). Hay dos sabores relevantes — **Activity Context** (vive y muere con esa Activity puntual, tiene noción de tema/UI) y **Application Context** (vive tanto como el proceso, sin ninguna referencia a una pantalla específica).

## 2. El problema que resuelve

Sin un ciclo de vida gestionado por el sistema, cada app tendría que decidir a mano cuándo liberar recursos de UI (cámara, sensores, listeners) y el sistema no podría destruir/recrear Activities de forma segura para liberar memoria cuando el usuario cambia de app, ni recrearlas ante un cambio de configuración (rotación de pantalla) sin perder todo el trabajo en curso. El lifecycle le da a la app un contrato predecible sobre cuándo la UI está visible, interactiva, o directamente destruida — sin eso, cualquier trabajo asincrónico (una corrutina, un listener, una animación) correría el riesgo de intentar tocar una UI que ya no existe, generando crashes o memory leaks.

## 3. Ejemplo mínimo comentado

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // de acá para abajo, todo lo que corre es Compose compartido de commonMain
        setContent {
            TimbaxTheme {
                TimbaxApp() // el único punto Android-only de toda la UI
            }
        }
    }
}
```

```kotlin
// Application Context: vive tanto como el proceso, sin atarse a ninguna Activity puntual
class TimbaxApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@TimbaxApplication) // Application Context, nunca Activity Context
            modules(appModule)
        }
    }
}
```

`MainActivity` es deliberadamente angosta: su única responsabilidad es el `setContent { }` inicial. `TimbaxApplication` es donde arranca Koin, porque los `single {}` que se declaran ahí (ver `koin_fundamentos_y_scopes.md`) necesitan un `Context` que sobreviva más que cualquier pantalla puntual.

## 4. Matriz de criterio

**Application Context vs. Activity Context:**
- Usar **Application Context** cuando: inicializás algo que vive más que cualquier Activity puntual — `androidContext()` de Koin, el engine de OkHttp para Ktor, el driver de SQLDelight. Todo lo declarado como `single {}` en Koin necesita Application Context, nunca Activity Context.
- Usar **Activity Context** cuando: necesitás algo atado a la UI/tema visual de esa pantalla puntual — inflar un layout XML legacy, mostrar un `Dialog`, lanzar un `Intent` que abre otra Activity.
- Trade-off / riesgo: guardar una Activity Context en algo de vida más larga que esa Activity misma es la causa #1 de memory leaks clásicos en Android (ver Caso trampa).

**Hookear lifecycle callbacks directamente (`onStart`/`onResume`) vs. usar Lifecycle-aware APIs:**
- Usar **Lifecycle-aware APIs** (`DisposableEffect` + `LocalLifecycleOwner`, `collectAsStateWithLifecycle()`, `viewModelScope`) cuando: el código vive dentro de Compose — es la recomendación oficial actual de Google: no hookear directamente `onStart`/`onResume` desde adentro de un composable.
- Hookear callbacks de Activity directamente cuando: es setup a nivel de toda la app que no tiene sentido modelar como efecto de Compose — el ejemplo canónico es el `setContent { }` de arriba, o `onNewIntent()` para deep links.
- Trade-off: los callbacks de Activity dan control total, pero mezclarlos con lógica que debería vivir en un efecto de Compose (`effects_guia_completa.md`) rompe el modelo de "composable como función reactiva a su `State`" y es fuente común de bugs de sincronización.

**Single-Activity architecture (lo que usa Compose/Timbax) vs. multi-Activity clásica:**
- Usar **single-Activity** cuando: el proyecto usa Compose — es el patrón recomendado oficialmente desde la introducción de Jetpack Compose, con la navegación entre "pantallas" resuelta dentro del mismo árbol de composables (`navegacion.md`), no con Activities separadas.
- Usar **multi-Activity** cuando: mantenimiento de código legacy con XML Views ya estructurado así, o un caso puntual donde una pantalla necesita un ciclo de vida completamente aislado (ej: el flujo de pago de un SDK de terceros que exige su propia Activity).
- Trade-off: single-Activity simplifica compartir estado entre pantallas (todo vive en el mismo proceso de Compose, sin serializar datos vía Intent extras), pero concentra toda la complejidad de navegación en un solo lugar que hay que diseñar bien desde el principio.

## 5. Caso trampa

Guardar una Activity Context en un `companion object` (el memory leak más citado de cualquier entrevista de Android):

```kotlin
// ❌ trampa: el companion object vive tanto como la clase — prácticamente toda la app —
// pero acá se le mete una referencia a una Activity puntual
class AnalyticsTracker private constructor(private val context: Context) {
    companion object {
        private var instance: AnalyticsTracker? = null
        fun init(activityContext: Context) {
            instance = AnalyticsTracker(activityContext) // 🚩 Activity Context en un singleton
        }
    }
}

// en MainActivity.onCreate():
AnalyticsTracker.init(this) // "this" acá es la Activity
```

El código compila y funciona en el primer uso. El problema aparece en la primera rotación de pantalla o navegación de vuelta: Android destruye la Activity original (`onDestroy`) para recrearla, pero el `companion object` de `AnalyticsTracker` sigue reteniendo una referencia fuerte a esa instancia ya destruida — el recolector de basura no puede liberarla porque un objeto de vida más larga (el singleton) todavía la referencia. Cada rotación deja una Activity "fantasma" completa en memoria, con todas las vistas y recursos que tenía cargados. La regla general: nunca guardar un `Context` de Activity en algo cuya vida excede la de esa Activity puntual — para eso existe `context.applicationContext`.

## 6. Conexión con Timbax

Timbax tiene exactamente una `MainActivity` del lado Android — es el único archivo verdaderamente "Android puro" del flujo de UI, y su única responsabilidad es el `setContent { }` que monta el árbol de Compose de `commonMain`. El resto del ciclo de vida se maneja indirectamente por abstracciones que el repo ya documenta: el `ViewModel` (`viewmodel.md`) sobrevive a la destrucción/recreación de esa Activity por su propio scope, `collectAsStateWithLifecycle()` (`collect_stateflow_en_compose.md`) pausa la colección del `StateFlow` cuando la Activity no está en foreground, y todos los `single {}` de Koin se inyectan a partir del `Application Context` de `TimbaxApplication` — nunca de un `Context` de Activity, precisamente para evitar el leak del Caso trampa. En otras palabras: buena parte de lo ya documentado en el repo (ViewModel, StateFlow con lifecycle, DisposableEffect) son abstracciones que existen justamente para que el día a día en Compose casi nunca requiera tocar estos callbacks de Activity a mano — pero un entrevistador SSR va a esperar que sepas qué hay debajo de esas abstracciones.