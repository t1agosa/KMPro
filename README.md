# KMP Reference — Manual de Referencia Senior

Manual de referencia personal para desarrollo Kotlin Multiplatform, construido a partir de experiencia real en producción — una app de delivery con seguimiento de pedidos (`Order`/`OrderItem`), sincronización offline y perfil de usuario en la nube: el historial de pedidos se sincroniza automáticamente entre dispositivos. No es un tutorial ni una guía de "cómo empezar" — es documentación orientada a **criterio**: cuándo elegir cada opción técnica y por qué, no solo qué es cada cosa.

## Por qué existe este repo

La mayoría de la documentación de KMP explica *qué es* cada pieza (Koin, DataStore, MVI). Lo que falta casi siempre es el criterio para decidir entre alternativas en un caso real, y las trampas conceptuales que aparecen recién cuando estás construyendo algo de verdad — o cuando tenés que auditar el código que una IA generó para vos. Este repo intenta llenar ese hueco: cada archivo cierra con un caso concreto tipo "un PO pide X" y un checklist explícito de qué revisar si esa implementación te la entregó una IA.

## Cómo está armado cada archivo

Los 90 archivos `.md` del repo siguen una estructura fija de 5 secciones:

1. **Mapa del flujo** — diagrama Mermaid del recorrido del dato/la pieza en el sistema.
2. **Qué es y cómo funciona** — definición + mecánica, apoyada en el diagrama.
3. **Cómo se ve en distintos contextos** — dos ejemplos cortos de apps genéricas, en prosa.
4. **Implementación real** — un caso concreto tipo "un PO pide X", con código completo.
5. **Buenas prácticas y errores comunes** — checklist explícito de qué auditar si el código lo entregó una IA.

Algunos archivos suman una **Sección 6+ "Profundización: [tema]"** cuando un concepto puntual necesita mecánica paso a paso que no entra en el formato checklist (ej. cómo se propaga `CancellationException` por la pila de llamadas, o el soporte KMP oficial de Navigation 3) — es la excepción, no la norma; el default sigue siendo 5 secciones.

## Stack de referencia

Kotlin Multiplatform · Clean Architecture (domain/data/di/presentation/ui) · MVI · Compose Multiplatform · DataStore · Firestore · Firebase (SDKs nativos) · Kermit · Koin · Ktor · Navigation 3.

## Cómo navegar el repo

Los packages están numerados para lectura tipo "bosque completo → detalle", pero cada archivo es autocontenido — se puede entrar directo a cualquiera sin leer los anteriores.

| Package | Contenido |
|---|---|
| `01_kotlin_fundamentals/` | Scope functions, null safety (+ `lateinit`), sealed classes, data class, generics, keywords reservadas (+ `inline`/`reified`/`crossinline`/`noinline`), extension functions, guía de identificación de toda forma de `fun` |
| `02_domain/` | Model, UseCases (reactivo vs. acción puntual), Repository contracts, Result pattern |
| `03_data/` | RepositoryImpl, DTOs/Mappers, Ktor remote, Retrofit (contraste), DataStore/Room/SQLDelight como opciones de storage local, Firestore remoto |
| `04_di/` | Koin (incl. `scope{}`), comparación con Dagger/Hilt y Kodein |
| `05_platform/` | expect/actual, expect/actual vs. interfaz + DI, source set hierarchy, Android lifecycle y fundamentals, WorkManager |
| `06_presentation_mvi/` | Contract (State/Event/Effect), ViewModel (incl. `SavedStateHandle`/proceso muerto), MVI vs. MVVM |
| `07_coroutines/` | Suspend/scope, Dispatchers, launch vs. async, SupervisorJob y excepciones |
| `08_flow/` | Flow básico, StateFlow, SharedFlow/Channel, operadores de Flow |
| `09_ui_compose/` | Composables/state hoisting, layouts, modifiers, recomposición, stability, remember, effects, collectAsState, listas lazy, Material3, CompositionLocal, resources multiplatform, navegación (+ Navigation 3), animaciones, accesibilidad, profiling/leaks |
| `10_ios_interop/` | Kotlin/Native, memory model, XCFramework/cinterop, interop inverso con Swift, Swift Export |
| `11_gradle/` | Modularización por feature, convention plugins, version catalog + BuildKonfig |
| `12_testing/` | Estrategia y prioridades de testing, fakes vs. mocks con Turbine, Espresso, Paparazzi, Detekt/JaCoCo |
| `13_git_y_equipo/` | Branching, merge vs. rebase, squash y conflictos, PRs/code review, conventional commits, GitHub Actions CI/CD, secretos y `.gitignore` |
| `14_criterio_y_decisiones/` | Traducir un pedido de PO a arquitectura, red flags en requisitos ambiguos, matriz de criterio para elegir stack |
| `15_ambientes_y_build_config/` | Flavors/schemes, BuildKonfig multi-ambiente, pipeline de promoción, CI/CD multi-ambiente, publicación en Play Console |
| `16_seguridad_mobile/` | TLS/certificate pinning, Play Integrity, app shielding (Appdome), gestión de SDKs de terceros |
| `17_observabilidad_produccion/` | Logging estructurado, telemetry/performance monitoring, tracking de eventos/analytics |