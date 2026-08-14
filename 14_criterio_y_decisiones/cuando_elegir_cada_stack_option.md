# Cuándo elegir cada opción de stack

## 1. Qué es

Es el criterio para decidir, entre las alternativas técnicas disponibles en KMP para un mismo problema (persistencia local, navegación, DI, networking engine), cuál conviene en función del contexto real del proyecto — y no de cuál es "la más nueva" o "la que usa Google". En KMP casi todo problema tiene 2 o 3 soluciones válidas; la habilidad senior no es conocer todas, es saber justificar por qué una encaja mejor que otra en *este* proyecto puntual.

## 2. El problema que resuelve

Sin un criterio explícito, las decisiones de stack se toman por dos caminos malos: "lo que ya sé" (cómodo, pero puede no encajar) o "lo que está de moda" (arriesgado, sin encaje probado). Ambos caminos ignoran la pregunta real: **¿el proyecto es KMP-first (multiplataforma desde el diseño) o Android-first con KMP como plus?** Esa sola pregunta resuelve la mayoría de las decisiones de stack de una — y no tenerla clara antes de elegir es lo que después obliga a migrar una librería entera a mitad de proyecto (ej: empezar con Room pensando "algún día agrego iOS" y después pelear con soporte KMP más nuevo y menos maduro que la alternativa que evitaba justamente ese problema desde el día uno).

## 3. Ejemplo mínimo comentado

```kotlin
// Pregunta base antes de elegir CUALQUIER pieza del stack:
// ¿Este proyecto tiene, o va a tener, targets reales fuera de Android
// (iOS, Desktop) en un plazo corto/mediano?

// Si la respuesta es SÍ (KMP-first, como Timbax):
// - Persistencia local → SQLDelight (pensada multiplataforma desde el día uno)
// - Navegación → Voyager o Decompose (nacieron para KMP, sin fricción histórica)
// - DI → Koin (100% Kotlin, sin generación de código atada a Android)
// - HTTP engine → Ktor con Darwin en iosMain, OkHttp en androidMain

// Si la respuesta es NO (Android-first, KMP como plus futuro):
// - Persistencia local → Room (KMP soportado desde 2.7, pero nació Android-only)
// - Navegación → Navigation Compose (menos fricción si el equipo ya lo conoce)
// - DI → Koin igual (Dagger/Hilt está atado al ciclo de vida de Android, encaja mal)
```

No hay una sola línea de código que resuelva esto — la decisión pasa por responder la pregunta base *antes* de tocar el `build.gradle.kts`.

## 4. Matriz de criterio

| Decisión | Elegí esto si... | Elegí la alternativa si... |
|---|---|---|
| **Persistencia local** | SQLDelight — proyecto KMP-first, o ya tenés iOS/Desktop en el roadmap | Room — proyecto Android-only hoy, equipo ya domina Room, iOS no está planeado a corto plazo |
| **Navegación** | Voyager/Decompose — KMP-first, necesitás manejo de ciclo de vida consistente entre plataformas | Navigation Compose — Android-first, el equipo ya conoce la API de Jetpack, fricción de adopción es prioridad |
| **DI** | Koin, casi siempre en KMP | Dagger/Hilt solo si el módulo es 100% Android sin intención real de compartirse |
| **HTTP engine (Ktor)** | OkHttp (Android) / Darwin (iOS) — plataforma nativa madura, mejor caché/interceptors/certificate pinning | CIO — solo cuando no hay alternativa nativa mejor (típicamente Desktop/JVM puro o backend) |
| **Branching (equipo)** | Trunk-based — equipo chico, releases frecuentes, CI/CD real | Git Flow — ciclos de release muy espaciados y controlados (ej: revisión manual larga en stores) |

**Trade-off real:** elegir la opción "más portable" (SQLDelight, Voyager) cuando el proyecto es 100% Android hoy agrega complejidad que no se paga todavía. Elegir la opción "más cómoda hoy" (Room, Navigation Compose) cuando ya sabés que va a haber iOS en 3 meses paga esa comodidad con una migración cara después. El criterio no es "cuál es mejor en abstracto" — es "cuál es mejor dado el horizonte real de este proyecto".

## 5. Caso trampa

"Vamos a usar Room porque es lo que promueve Google y ya sabemos usarlo, después si hace falta iOS ya lo migramos." Suena razonable — Room ya soporta KMP desde la 2.7 — pero la trampa está en subestimar el costo de "migrar después": para ese momento ya hay decenas de queries, DAOs y migraciones de schema escritas contra la API de Room, y aunque Room-KMP funcione, la madurez y el ecosistema de tooling alrededor de SQLDelight en multiplataforma (generación de código type-safe pensada desde el diseño, no adaptada después) sigue siendo mayor. La decisión "elijo Room porque ya lo sé" es válida si el horizonte es realmente Android-only — pero si en el fondo ya se sabe que iOS viene, esa comodidad de corto plazo es la que después se paga cara. El error no es elegir Room — es elegirlo sin haber respondido honestamente la pregunta base de la sección 2.

## 6. Conexión con arquitectura real (Timbax)

Timbax es el ejemplo del camino "KMP-first" resuelto con la ruta "Estándar" del `00_MASTER_PLAN`: SQLDelight para persistencia, Ktor con engines nativos por plataforma, Koin para DI, y Firebase vía GitLive SDK (la variante multiplatform, no el SDK nativo de Android) — cada elección responde a la misma pregunta base: el proyecto nació con la intención de ser realmente multiplataforma, así que cada pieza del stack se eligió pensando en ese horizonte desde el día uno, no parcheada después.