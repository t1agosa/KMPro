# Detekt + JaCoCo (Análisis Estático y Coverage)

## 1. Qué es

Dos herramientas que automatizan dos preguntas distintas sobre la calidad del código, ninguna de las cuales depende de correr tests:

- **Detekt**: analizador estático para Kotlin. Lee el código fuente (sin ejecutarlo) y reporta *code smells* — funciones demasiado largas, complejidad ciclomática alta, nombres poco claros, imports sin usar — según un set de reglas configurables (`detekt.yml`). Puede además envolver `ktlint` (vía el módulo `detekt-formatting`) para chequear estilo/formato en la misma pasada.
- **JaCoCo**: herramienta de *code coverage*. A diferencia de Detekt, sí necesita que los tests corran — instrumenta el bytecode para registrar qué líneas/branches se ejecutaron durante la suite de tests, y produce un reporte con el porcentaje real de código cubierto.

Detekt pregunta "¿este código está bien escrito?" sin correrlo jamás. JaCoCo pregunta "¿mis tests realmente tocan este código?" y solo puede responder después de ejecutarlos.

## 2. El problema que resuelve

Sin Detekt, los *code smells* (funciones gigantes, complejidad innecesaria, convenciones de nombres inconsistentes) se atrapan únicamente en code review humano — algo inconsistente, que depende de quién revisa ese día, y que no escala a medida que el equipo crece. Sin JaCoCo, "tenemos tests" es una afirmación sin evidencia objetiva: no hay forma de distinguir entre una suite que realmente ejerce el 80% de la lógica de negocio y una con 3 tests que siempre pasan por el mismo camino feliz, dejando ramas de error completamente sin cubrir. Ambas herramientas automatizan un gate de calidad que de otra forma dependería enteramente del criterio (y el tiempo disponible) de quien revisa el Pull Request.

## 3. Ejemplo mínimo comentado

```kotlin
// build.gradle.kts — Detekt
plugins {
    id("io.gitlab.arturbosch.detekt") version "1.23.6"
}

detekt {
    buildUponDefaultConfig = true // arranca con las reglas default de Detekt como base
    config.setFrom("$projectDir/config/detekt.yml") // reglas propias del proyecto, por encima del default
    baseline = file("$projectDir/config/baseline.xml") // suprime deuda YA existente, no deuda nueva
}
```

```kotlin
// build.gradle.kts — JaCoCo, sobre un módulo Android con Kotlin
apply(plugin = "jacoco")

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest") // corre primero los tests, JaCoCo mide sobre esa ejecución
    reports {
        xml.required.set(true)  // formato que consumen herramientas de CI (SonarQube, etc.)
        html.required.set(true) // formato navegable para revisar a mano
    }
}
```

`baseline.xml` es la pieza clave para adoptar Detekt en un proyecto que ya tiene código: en vez de forzar arreglar cientos de *findings* existentes de una sola vez, el baseline los "congela" como conocidos, y Detekt solo falla el build ante *código nuevo* que viole las reglas — la deuda vieja queda documentada, no bloqueando.

## 4. Matriz de criterio

**Detekt local (IDE/pre-commit) vs. en CI:**
- Correrlo en el IDE/pre-commit cuando: querés feedback inmediato mientras escribís código, antes de siquiera abrir el PR — reduce la fricción de descubrir violaciones recién en CI, minutos después.
- Correrlo en CI (bloqueante) cuando: es la garantía real de que nada con violaciones nuevas llega a `main` — el pre-commit es una ayuda, pero nunca reemplaza el gate obligatorio en el pipeline, porque es fácil saltearse un hook local.
- Trade-off: ninguno real — ambos deberían coexistir, no es una decisión de "uno u otro".

**`baseline.xml` vs. arreglar todo antes de habilitar Detekt:**
- Usar `baseline` cuando: el proyecto ya tiene una cantidad significativa de código y arreglar todos los *findings* de una sentada no es viable — permite adoptar la herramienta ya, sin bloquear el resto del equipo por semanas.
- Arreglar todo de entrada cuando: el proyecto es nuevo o chico — no vale la pena introducir un baseline si el volumen de *findings* iniciales es manejable en una tarde.
- Trade-off: el baseline es pragmático pero puede convertirse en una alfombra debajo de la cual esconder deuda indefinidamente si nadie revisita el archivo — conviene tratarlo como una lista a reducir con el tiempo, no como permanente.

**Cobertura de JaCoCo como gate obligatorio (ej: "no bajar de X%") vs. como métrica informativa:**
- Usar como gate obligatorio cuando: el equipo ya tiene disciplina de testing establecida y el número sirve para prevenir regresiones de cobertura, no para perseguir un número alto por sí mismo.
- Usar como métrica informativa (sin bloquear el build) cuando: el proyecto está introduciendo cultura de testing recién — un gate estricto desde el día uno incentiva tests superficiales que solo "tocan la línea" para subir el número, sin verificar nada real.
- Trade-off: un gate de cobertura mal calibrado es peor que no tenerlo — un equipo bajo presión de un número puede escribir tests vacíos de valor solo para pasar el gate, lo que en la práctica reduce la calidad real de la suite mientras el dashboard muestra "mejora".

**Detekt solo vs. Detekt + `detekt-formatting` (wrapper de ktlint):**
- Sumar `detekt-formatting` cuando: además de code smells, querés que la misma herramienta chequee estilo/formato (indentación, espacios, orden de imports) — evita mantener dos herramientas separadas para dos aspectos relacionados.
- Mantener Detekt solo cuando: el formato ya se resuelve con otra herramienta (el formatter nativo de Android Studio, o `ktlint` corriendo independiente) — no hay necesidad real de duplicar esa responsabilidad.

**Detekt/JaCoCo standalone vs. integrados en SonarQube:**
- Usar Detekt + JaCoCo standalone (solo en CI, sin SonarQube) cuando: el proyecto es chico o personal (como Timbax) — los reportes de cada herramienta alcanzan por sí solos, y sumar un servidor de SonarQube sería complejidad sin beneficio real a esa escala.
- Integrar ambos en SonarQube cuando: hay varios proyectos/módulos y un equipo que necesita una vista centralizada — SonarQube no reemplaza a Detekt ni a JaCoCo, los **consume**: tiene su propio analizador de Kotlin (bugs, vulnerabilidades, code smells) y además soporta oficialmente importar el reporte XML de Detekt (`sonar.kotlin.detekt.reportPaths`) y el de JaCoCo, mostrando todo junto en un solo dashboard con historial y Quality Gates configurables (ej: "cobertura de código nuevo no puede bajar de X%").
- Trade-off: SonarQube agrega una pieza de infraestructura más para mantener (servidor propio o SonarCloud), a cambio de una vista consolidada y tendencias en el tiempo que ni Detekt ni JaCoCo dan por separado.
## 5. Caso trampa

Confiar ciegamente en el porcentaje que reporta JaCoCo para funciones que usan `inline`, sin entender que Kotlin y JaCoCo tuvieron (y en casos puntuales siguen teniendo) fricción real midiendo ese caso:

```kotlin
// commonMain — una función inline, como cualquier scope function de Kotlin
inline fun <T> T.applyIfValid(condition: Boolean, block: T.() -> Unit): T {
    if (condition) block()
    return this
}

// test que SÍ ejercita la función
@Test
fun `applyIfValid ejecuta el block cuando condition es true`() {
    val result = Player(id = "1", name = "Tiago", score = 0)
        .applyIfValid(condition = true) { /* ... */ }
    // el test pasa, la función se ejecutó de verdad
}
```

Durante años, JaCoCo reportó funciones `inline` de Kotlin como 0% cubiertas incluso cuando estaban efectivamente testeadas — porque el compilador de Kotlin, al inlinear la función en cada call site, deja el bytecode en una forma que JaCoCo no siempre asocia correctamente con la línea original. Versiones recientes de JaCoCo (0.8.13, 2025) mejoraron esto explícitamente — agregando cálculo de cobertura para funciones `inline`, incluidas las que tienen parámetros `reified`, y filtrando bytecode generado por el compiler plugin de Compose para no ensuciar el número — pero a la fecha siguen reportándose casos puntuales donde el número sale inflado o directamente en cero según dónde vive el call site. La trampa concreta: ver "0% de cobertura" en una función `inline` (las scope functions `let`/`apply`/`run`/`also`, documentadas en `scope_functions.md`, son inline por definición, y Compose usa `inline` extensivamente en su propio DSL) y asumir que significa "no está testeada", cuando puede ser una limitación de medición, no un hueco real de testing. La verificación correcta no es confiar en el número aislado — es correlacionarlo con la lista real de tests que existen para esa función antes de decidir que falta cobertura.

## 6. Conexión con Timbax

Detekt encaja de forma directa en el domain de Timbax: una regla como complejidad ciclomática alta señalaría, por ejemplo, si el cálculo de puntaje de una ronda de Chinchón (con sus reglas de bonificación, corte, y penalizaciones) creciera hasta un punto donde un solo `UseCase` se vuelve difícil de leer — una señal objetiva de que esa lógica merece descomponerse, en vez de esperar a que alguien lo note en code review. JaCoCo, en cambio, aplica con más matiz: el domain de Timbax (`UseCase`s, mappers) es justamente donde el patrón de este repo prioriza testear primero (`estrategia_y_prioridades.md`), y ahí la mayoría del código no es `inline`, así que el número de JaCoCo es confiable sin mayor cuidado. Donde el caso trampa de este archivo sí aplicaría es en el código de Compose UI — que en Timbax, siguiendo la misma estrategia de prioridades, casi no se testea con tests automatizados de todos modos, así que en la práctica la limitación de JaCoCo con código `inline` termina siendo más una curiosidad a conocer para una entrevista que un problema real del día a día del proyecto.