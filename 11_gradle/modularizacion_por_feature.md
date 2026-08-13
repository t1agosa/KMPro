# Modularización por feature

## 1. Qué es

Dividir el código de un proyecto KMP en módulos Gradle independientes en vez de tener todo (o casi todo) en un único módulo `:shared` o `:composeApp`. Típicamente se separa en tres tipos de módulo: `:core:*` (código transversal — network, database, design-system), `:feature:*` (una feature completa, con sus propias sub-capas domain/data/presentation/ui), y el módulo `app` final que ensambla todo.

## 2. El problema que resuelve

En un proyecto chico, tener todo en `:shared` funciona bien. Pero a medida que el proyecto crece aparecen dos problemas concretos:

- **Build times.** Gradle recompila el módulo completo ante cualquier cambio, aunque sea en una sola pantalla. En un módulo gigante, eso significa esperar minutos por un cambio de una línea.
- **Sin barrera real entre features.** Nada impide que el código de "scoreboard" importe directamente una clase `internal` de "players" porque, al vivir en el mismo módulo, técnicamente pueden verse entre sí. La separación es solo de carpetas, no del compilador.

Con módulos separados, Gradle solo recompila lo que cambió (y lo que depende de eso), y el compilador impide literalmente que una feature acceda a los internals de otra — es una barrera forzada, no una convención que alguien puede romper sin darse cuenta.

## 3. Ejemplo mínimo comentado

```kotlin
// settings.gradle.kts — declarás cada módulo como proyecto Gradle independiente
include(":core:network")
include(":core:database")
include(":core:design-system")
include(":feature:players")
include(":feature:scoreboard")
include(":composeApp")
```

```kotlin
// feature/players/build.gradle.kts
plugins {
    id("org.jetbrains.kotlin.multiplatform")
    id("org.jetbrains.compose")
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()

    sourceSets {
        commonMain.dependencies {
            // depende de :core, nunca de otro :feature directamente
            implementation(project(":core:network"))
            implementation(project(":core:design-system"))
        }
    }
}
```

```kotlin
// dentro de feature/players, una clase que NO se expone fuera del módulo
internal class PlayerRepositoryImpl(
    private val dao: PlayerDao
) : PlayerRepository
// :feature:scoreboard NO puede importar esto ni por accidente —
// el compilador lo bloquea, no depende de que nadie lo intente
```

### Ejemplo a escala: una app bancaria

El caso de Timbax con 2-3 features se siente chico para justificar modularizar. Una app bancaria es el ejemplo opuesto — ahí la modularización deja de ser una decisión estética y se vuelve casi obligatoria.

```
banco-app/
├── build-logic/                    (convention plugins)
├── core/
│   ├── network/                    (cliente Ktor configurado, interceptors de auth)
│   ├── database/                   (SQLDelight, cifrado local)
│   ├── design-system/              (componentes visuales de marca, tema)
│   ├── auth-session/                (token actual, refresh, logout — lo usan TODAS las features)
│   └── analytics/                  (tracking transversal)
├── feature/
│   ├── login/                      (login, biometría, recuperación de clave)
│   ├── home-dashboard/             (resumen de cuentas, accesos rápidos)
│   ├── transfers/                  (transferencias entre cuentas y a terceros)
│   ├── cards/                      (tarjetas: bloqueo, límites, resumen)
│   ├── investments/                (plazo fijo, fondos)
│   ├── loans/                      (préstamos, simulador de cuotas)
│   └── support-chat/               (chat de soporte)
├── androidApp/                     (ensambla todo, entrypoint Android)
└── iosApp/                         (proyecto Xcode, consume el framework)
```

Por qué este caso es distinto al de un card game:

- **Equipos reales por feature.** En un banco, es común que el equipo de "transfers" y el de "investments" sean literalmente personas distintas, a veces hasta con ciclos de release desacoplados. Sin límites de módulo forzados por el compilador, dos equipos grandes tocando el mismo módulo gigante es garantía de conflictos de merge constantes.
- **Compliance y superficie de ataque.** Un bug en `:feature:support-chat` no debería poder, ni por accidente de import, tocar código de `:feature:transfers`. La barrera de módulo ayuda a que un audit de seguridad pueda razonar "qué módulos tocan dinero" de forma explícita, en vez de tener que revisar un monolito completo.
- **`:core:auth-session` como ejemplo de core bien puesto.** El token de sesión y su refresh los necesitan *todas* las features (no tiene sentido que "cards" y "loans" reimplementen su propio manejo de sesión) — es el ejemplo típico de algo que sí pertenece a `:core` y no a una feature específica, distinto del caso trampa de la sección 5.
- **Build times con impacto de negocio real.** Si un dev de `:feature:loans` cambia el simulador de cuotas, no tiene sentido esperar que recompile `:feature:investments` — en un proyecto de este tamaño (fácilmente 15-20 módulos), la diferencia entre modularizar o no es la diferencia entre un build de segundos y uno de varios minutos, multiplicado por cada dev, cada día.

## 4. Matriz de criterio

**Modularizar cuando:**
- El proyecto ya tiene 3+ features con lógica propia sustancial (no una pantalla de 20 líneas).
- Los build times empiezan a doler en el día a día (más de unos segundos por cambio chico).
- Hay o va a haber más de una persona tocando el proyecto en paralelo.
- Necesitás que un equipo/persona pueda trabajar en una feature sin arriesgar romper otra por accidente.

**NO modularizar (todavía) cuando:**
- El proyecto es chico o un prototipo — la modularización agrega overhead de configuración (cada módulo necesita su propio `build.gradle.kts`, sourceSets, targets) que no se paga sola si hay 2 pantallas.
- Sos el único dev y el proyecto entero compila en segundos igual.
- Timbax hoy, por ejemplo: con el tamaño actual, un único módulo compartido sigue siendo la opción correcta — modularizar ahora sería sobre-ingeniería sin beneficio real todavía.

**Trade-off real:** ganás build times + encapsulación forzada, pero pagás con más configuración repetida por módulo (mitigable con convention plugins, que es justo el próximo archivo) y con la fricción de decidir constantemente "¿esto va en `:core` o es específico de esta feature?" — esa decisión mal tomada genera módulos `:core` que terminan siendo un cajón de sastre.

## 5. Caso trampa

Te preguntan: *"¿`:feature:players` puede depender directamente de `:feature:scoreboard` si necesita mostrar el nombre del jugador ganador?"*

La respuesta obvia parece "sí, totalmente, ya que ambos son módulos del mismo proyecto y Gradle lo permite sin problema técnico". Pero es la respuesta incorrecta: permitir dependencias cruzadas entre módulos `:feature:*` hermanos reintroduce exactamente el acoplamiento que la modularización busca evitar — si mañana `:feature:scoreboard` cambia su modelo interno, rompés `:feature:players` sin que sea obvio por qué. Lo correcto es que lo compartido (en este caso, quizás un modelo `Player` básico o un caso de uso `GetWinnerUseCase`) viva en `:core` (o en un módulo intermedio tipo `:feature:players-api` si el dato es más específico), y que ambas features dependan de ahí — nunca una feature de otra feature directamente.

## 6. Conexión con Timbax

Timbax hoy vive en un único módulo compartido, y es la decisión correcta para su tamaño actual — no hay necesidad de modularizar todavía. Pero el ejercicio de pensarlo sirve como caso de estudio: si Timbax creciera con más juegos (Chinchón, Truco, Generala) cada uno con su propia lógica de puntaje y pantallas, ahí sí el patrón `:core:scoring` (compartido) + `:feature:chinchon`, `:feature:truco`, `:feature:generala` (cada uno independiente) empezaría a pagarse solo — porque hoy, sin modularizar, cualquier cambio en la lógica de un juego obliga a recompilar el módulo entero, aunque los otros juegos no se hayan tocado.