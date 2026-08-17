# KMP Reference — Manual de Referencia Senior

Manual de referencia personal para desarrollo Kotlin Multiplatform, construido a partir de experiencia real en producción (**Timbax**, app de scoring para juegos de cartas argentinos como Truco, Generala y Chinchón, con perfil de usuario sincronizado en la nube: una partida ranked jugada en cualquier dispositivo se sincroniza automáticamente a la cuenta con una validación de seguridad). No es un tutorial ni una guía de "cómo empezar" — es documentación orientada a **criterio**: cuándo elegir cada opción técnica y por qué, no solo qué es cada cosa.

## Por qué existe este repo

La mayoría de la documentación de KMP explica *qué es* cada pieza (Koin, SQLDelight, MVI). Lo que falta casi siempre es el criterio para decidir entre alternativas en un caso real, y las trampas conceptuales que aparecen recién cuando estás construyendo algo de verdad. Este repo intenta llenar ese hueco: cada archivo conecta el concepto con una decisión real tomada en Timbax.

## Cómo está armado cada archivo

Los ~88 archivos `.md` del repo siguen una estructura fija de 6 secciones:

1. **Qué es** — definición concreta, sin rodeos.
2. **El problema que resuelve** — qué pasaba antes / qué pasa si no existiera.
3. **Ejemplo mínimo comentado** — código real, mínimo posible.
4. **Matriz de criterio** — usar cuando / NO usar cuando / trade-off real.
5. **Caso trampa** — situación ambigua donde la respuesta obvia es incorrecta.
6. **Conexión con Timbax** — cómo se aplica (o por qué deliberadamente no se aplica) en la app real.

## Stack de referencia (Timbax)

Kotlin Multiplatform · Clean Architecture (domain/data/di/presentation/ui) · MVI · Compose Multiplatform · Firebase (GitLive SDK) · Kermit · SQLDelight · Koin · Ktor · OkHttp/Darwin engines.

## Cómo navegar el repo

Los packages están numerados para lectura tipo "bosque completo → detalle", pero cada archivo es autocontenido — se puede entrar directo a cualquiera sin leer los anteriores.

| Package | Contenido |
|---|---|
| `01_kotlin_fundamentals/` | Scope functions, null safety, sealed classes, data class, generics, keywords |
| `02_domain/` | Model, UseCases, Repository contracts, Result pattern |
| `03_data/` | RepositoryImpl, DTOs/Mappers, Ktor remote, Retrofit (contraste), storage local, engines por plataforma |
| `04_di/` | Koin, comparación con Dagger/Hilt y Kodein |
| `05_platform/` | expect/actual, source set hierarchy, Android lifecycle y fundamentals |
| `06_presentation_mvi/` | Contract (State/Event/Effect), ViewModel, MVI vs MVVM |
| `07_coroutines/` | Suspend/scope, Dispatchers, launch vs async, SupervisorJob |
| `08_flow/` | Flow, StateFlow, SharedFlow/Channel |
| `09_ui_compose/` | Layouts, modifiers, listas, Material3, recomposición, stability, effects, navegación, animaciones, accesibilidad, profiling/leaks, Navigation 3 |
| `10_ios_interop/` | Kotlin/Native, XCFramework, memory model, interop con Swift, Swift Export |
| `11_gradle/` | Modularización, convention plugins, version catalog, BuildKonfig |
| `12_testing/` | Estrategia de testing, fakes vs mocks, Turbine, Espresso, Paparazzi, Detekt/JaCoCo |
| `13_git_y_equipo/` | Branching, merge vs rebase, PRs, conventional commits, CI/CD |
| `14_criterio_y_decisiones/` | Traducir requisitos de PO, matriz de stack, red flags |
| `15_ambientes_y_build_config/` | Flavors/schemes, BuildKonfig multi-ambiente, pipelines, CI/CD multi-ambiente, publicación en Play Console |
| `16_seguridad_mobile/` | TLS/pinning, Play Integrity, app shielding, SDKs de terceros |
| `17_observabilidad_produccion/` | Logging estructurado, telemetry/performance, tracking de eventos |