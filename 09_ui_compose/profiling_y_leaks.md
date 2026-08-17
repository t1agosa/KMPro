# Profiling y Memory Leaks

## 1. Qué es

Profiling es el proceso de medir dónde efectivamente se gasta tiempo/CPU/memoria en una app en runtime, en vez de asumirlo. En el ecosistema Compose hay dos herramientas centrales para esto: los **Compose Compiler Metrics/Reports** — archivos de texto generados en build-time que listan, por cada composable, si es "restartable" y "skippable" (y por qué no, si no lo es) — y el **Layout Inspector** de Android Studio, que muestra en runtime cuántas veces recompuso efectivamente cada composable mientras interactuás con la app. Memory leaks (fugas de memoria) es un problema relacionado pero distinto: objetos que deberían haber sido liberados por el recolector de basura (porque su Activity/Fragment/ViewModel ya se destruyó) pero siguen retenidos porque algo en la app todavía los referencia — la herramienta estándar para detectarlos es **LeakCanary**, de Square.

## 2. El problema que resuelve

"Se siente lento" es una percepción, no un diagnóstico. Sin profiling, es imposible saber si el problema es recomposición excesiva (ver `recomposicion.md` y `stability.md`), un layout costoso de medir/dibujar, o simplemente trabajo síncrono bloqueando el hilo principal — y sin esa distinción, cualquier intento de "optimizar" es a ciegas. Los memory leaks son todavía más traicioneros: no generan ningún síntoma inmediato (la app sigue funcionando "normal"), la memoria retenida crece de a poco con cada Activity/pantalla que se destruye sin liberarse del todo, y el síntoma final — un crash por `OutOfMemoryError` — aparece mucho después y lejos del código que realmente causó el leak, casi imposible de rastrear a mano.

## 3. Ejemplo mínimo comentado

```kotlin
// build.gradle.kts del módulo: habilita los reportes del compilador de Compose
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_compiler")
}
```

```kotlin
// build.gradle.kts: LeakCanary solo en debug, cero código extra
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
    // no hace falta tocar Application.onCreate() ni ningún otro archivo
}
```

Correr `./gradlew assembleRelease` con `composeCompiler` configurado genera, por módulo, un archivo `<modulo>-composables.txt` que lista cada composable con sus atributos (`restartable`, `skippable`, o no) y un archivo `<modulo>-classes.txt` con la estabilidad de cada clase — la fuente de verdad "estática" para todo lo documentado en `stability.md`, ahora medible en vez de intuido. Es importante correrlo sobre un build de release: los builds de debug desactivan algunas optimizaciones del compilador y los reportes salen distorsionados. LeakCanary, por su lado, no necesita ningún código adicional — se auto-instala (vía `ContentProvider`) y empieza a vigilar automáticamente `Activity`, `Fragment` y `ViewModel` apenas está en el classpath de `debug`.

## 4. Matriz de criterio

**Compose Compiler Reports (build-time) vs. Layout Inspector (runtime):**
- Usar Compiler Reports cuando: querés un inventario completo y objetivo de qué composables/clases del proyecto son estables/skippables — es la forma de auditar toda la codebase de una sola vez, sin tener que interactuar con la app.
- Usar Layout Inspector cuando: ya identificaste una pantalla puntual con jank y necesitás ver, en vivo, cuántas veces recompone cada composable mientras interactuás — te dice qué está pasando de verdad, no solo qué podría pasar.
- Trade-off: Compiler Reports es exhaustivo pero estático (no distingue "esto recompone 50 veces por segundo" de "esto nunca recompone en la práctica"); Layout Inspector es preciso sobre lo que ocurre pero solo cubre la sesión de debug puntual que estás corriendo.

**Cuándo correr un audit de performance:**
- Correrlo cuando: hay jank visible reportado (frames perdidos, scroll entrecortado), antes de un release grande, o como parte de la preparación de una entrevista técnica (mostrar que sabés leer estos reportes es justamente lo que un entrevistador SSR/senior espera).
- NO perseguir 100% de skippability en todo el proyecto sin evidencia de un problema real — como ya se documentó en `stability.md`, es optimización, no corrección; el tiempo tiene mejor retorno resolviendo los 2-3 composables que el reporte marca como "restartable pero no skippable" dentro de una `LazyColumn` grande, que auditando pantallas estáticas que nunca recomponen.

**LeakCanary en debug vs. no tener nada:**
- Usar cuando: siempre, en cualquier proyecto Android en desarrollo activo — el costo de setup es una línea de Gradle (`debugImplementation`), y LeakCanary corre completamente aislado de los builds de release (nunca termina en el APK que llega a Play Store).
- Trade-off: ninguno real para el caso general — la única razón legítima para no tenerlo sería un proyecto tan chico/descartable que ni siquiera vale la pena instrumentarlo, algo raro en un proyecto en producción como Timbax.

**Baseline Profiles / Macrobenchmark (mención, fuera del alcance de este archivo):**
- Usar cuando: además de recomposición y leaks, interesa medir y mejorar métricas más "macro" — tiempo de arranque en frío, jank durante un scroll específico — con benchmarks automatizados y reproducibles en CI.
- NO son el foco de este archivo porque atacan un problema distinto (tiempo de arranque/scroll medido end-to-end) al de recomposición puntual o leaks de memoria — se mencionan acá solo como el siguiente paso lógico una vez que el proyecto ya tiene profiling y detección de leaks resueltos.

## 5. Caso trampa

Registrar un listener de Android dentro de un `LaunchedEffect`, asumiendo que la cancelación de la corrutina al salir de composición alcanza para limpiarlo:

```kotlin
// ❌ trampa: LaunchedEffect cancela la corrutina, pero eso no desregistra el listener
@Composable
fun PlayerListScreen(viewModel: PlayersViewModel, connectivityManager: ConnectivityManager) {
    LaunchedEffect(Unit) {
        val listener = object : NetworkStateListener {
            override fun onChange(isConnected: Boolean) {
                viewModel.onEvent(PlayersEvent.OnConnectivityChanged(isConnected))
            }
        }
        connectivityManager.registerListener(listener) // 🚩 nunca se desregistra
    }
}
```

`LaunchedEffect` cancela la corrutina cuando el composable sale de composición — pero cancelar una corrutina solo limpia lo que la corrutina misma controla (otras corrutinas hijas, un `delay()`, la colección de un `Flow`). No tiene ningún poder sobre un objeto que un sistema externo a las coroutines — acá, `ConnectivityManager`, un servicio atado al ciclo de vida de la `Application`, no de la pantalla — sigue reteniendo por su cuenta. El `listener` (y todo lo que capturó por closure, incluido potencialmente el `viewModel`) queda vivo indefinidamente, mucho después de que el usuario salió de esa pantalla. LeakCanary lo va a reportar como un `ViewModel` o una `Activity` retenida, y la traza de referencias termina en este listener nunca desregistrado — encontrado días después, lejos del código que lo causó.

```kotlin
// ✅ correcto: DisposableEffect obliga a declarar la limpieza explícita
DisposableEffect(Unit) {
    val listener = object : NetworkStateListener {
        override fun onChange(isConnected: Boolean) {
            viewModel.onEvent(PlayersEvent.OnConnectivityChanged(isConnected))
        }
    }
    connectivityManager.registerListener(listener)
    onDispose {
        connectivityManager.unregisterListener(listener) // limpieza explícita y obligatoria
    }
}
```

La señal de alarma: cualquier callback/listener de una API que no sea coroutine-based (no es una `suspend fun` ni expone un `Flow`) registrado dentro de un `LaunchedEffect` es candidato a este leak — la corrección es usar `DisposableEffect`, cuyo `onDispose { }` obligatorio existe exactamente para forzar esta limpieza (ya documentado en `effects_guia_completa.md`).

## 6. Conexión con Timbax

Timbax, siendo una app chica pero real en producción, es un buen caso para no sobre-invertir: correr los Compose Compiler Reports una vez por release grande (no en cada PR) para confirmar que las pantallas con listas (el historial de partidas, el listado de jugadores — ver `listas_lazy.md`) mantienen sus composables skippables, y tener LeakCanary como `debugImplementation` permanente durante el desarrollo activo — su costo de setup es nulo y su beneficio (detectar un leak antes de que un usuario real lo sufra como lentitud o crash tras minutos de uso) es alto para una app pensada para sesiones largas de juego. La combinación de ambas herramientas es la respuesta concreta al gap identificado en el análisis de mercado del repo (`analisis.md`): "optimización de recomposición en Compose, memory leaks (LeakCanary lo pide EPAM)" — ya no es una intención declarada, es una práctica documentada con ejemplos reales.