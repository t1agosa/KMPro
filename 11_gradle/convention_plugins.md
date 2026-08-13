# Convention plugins

## 1. Qué es

Plugins Gradle custom (típicamente ubicados en un módulo `build-logic` o `buildSrc`) que centralizan configuración repetida entre módulos: sourceSets, targets (Android/iOS/Desktop), versiones de Compose, dependencias comunes. En vez de que cada `build.gradle.kts` de cada feature reescriba 40 líneas de configuración de targets, aplica un solo plugin propio (`id("convention.kmp-feature")`) que ya trae todo eso resuelto.

## 2. El problema que resuelve

Modularizar (ver archivo anterior) trae un costo colateral: cada módulo nuevo necesita su propio `build.gradle.kts` con targets (`androidTarget()`, `iosX64()`, `iosArm64()`, `iosSimulatorArm64()`), configuración de Compose, y dependencias base. En un proyecto con 3-4 módulos, copiar y pegar esa configuración en cada uno es molesto pero tolerable. En un proyecto con 15-20 módulos (como el ejemplo bancario del archivo anterior), se vuelve un problema real:

- **Duplicación masiva.** El mismo bloque de configuración de targets repetido en cada `build.gradle.kts` — si hay que agregar un target nuevo (por ejemplo, sumar soporte Wasm), hay que tocar 20 archivos uno por uno.
- **Inconsistencia silenciosa.** Alguien crea un módulo nuevo copiando el `build.gradle.kts` de otro módulo más viejo, y arrastra configuración desactualizada o mal ajustada sin darse cuenta — no hay una única fuente de verdad de "así se configura un módulo feature en este proyecto".
- **Fricción para crear módulos nuevos.** Si crear un módulo implica escribir 40 líneas de Gradle a mano, eso desalienta la modularización fina — la gente termina metiendo cosas de más en un módulo existente en vez de crear uno nuevo, justo lo contrario de lo que se buscaba en el archivo anterior.

## 3. Ejemplo mínimo comentado

```kotlin
// build-logic/src/main/kotlin/convention.kmp-feature.gradle.kts
// Este archivo ES el plugin — su nombre de archivo define el id del plugin
plugins {
    id("org.jetbrains.kotlin.multiplatform")
    id("org.jetbrains.compose")
}

kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        commonMain.dependencies {
            implementation(project(":core:network"))
            implementation(project(":core:design-system"))
        }
    }
}
```

```kotlin
// build-logic/build.gradle.kts — declara que build-logic es, en sí,
// un proyecto Gradle que compila plugins (no una app)
plugins {
    `kotlin-dsl`
}
```

```kotlin
// feature/players/build.gradle.kts — así de simple queda cada módulo
// toda la configuración de targets/Compose/dependencias base ya viene del plugin
plugins {
    id("convention.kmp-feature")
}

kotlin {
    sourceSets {
        commonMain.dependencies {
            // acá solo van las dependencias ESPECÍFICAS de esta feature
            implementation(project(":core:database"))
        }
    }
}
```

## 4. Matriz de criterio

**Usar convention plugins cuando:**
- Ya tenés 4-5+ módulos con configuración de targets/Compose repetida — el punto de quiebre suele ser cuando cambiar algo de esa config a mano implica tocar más de 2-3 archivos.
- Vas a seguir creando módulos nuevos con cierta frecuencia (proyecto en crecimiento activo, no uno ya cerrado).
- Hay más de una persona creando módulos — la consistencia forzada por el plugin evita que cada dev configure su módulo "a su manera".

**NO usar (todavía) cuando:**
- Tenés 1-3 módulos — el costo de mantener un `build-logic` propio (otro proyecto Gradle dentro del proyecto, con su propia curva de setup) no se paga con tan poca duplicación.
- La configuración entre módulos ya es genuinamente distinta caso a caso (poco realista en la práctica, pero si pasara, forzar un plugin común generaría más parámetros condicionales que ahorro real).

**Trade-off real:** ganás consistencia y una única fuente de verdad para la configuración de módulos, pero pagás con una capa extra de indirección — alguien nuevo en el proyecto tiene que entender que `id("convention.kmp-feature")` no es un plugin de JetBrains sino uno propio, y para ver "qué hay realmente configurado" tiene que ir a mirar `build-logic/` en vez de tenerlo todo a la vista en el `build.gradle.kts` del módulo. Es la misma lógica que un `BaseViewModel` abstracto: centralizar ahorra repetición, pero agrega un nivel de indirección que hay que conocer.

## 5. Caso trampa

Te preguntan: *"Si dos features necesitan casi la misma configuración pero una tiene una dependencia extra muy específica (por ejemplo, `:feature:cards` necesita una librería de escaneo de tarjetas que ninguna otra feature usa), ¿esa dependencia va dentro del convention plugin `convention.kmp-feature` para que esté disponible 'por las dudas'?"*

La respuesta que parece práctica es "sí, así queda disponible en todos lados y no hay que tocar nada si otra feature la necesita después". Es la respuesta incorrecta: meter dependencias específicas de una sola feature dentro de un convention plugin compartido rompe el propósito del plugin (configuración genuinamente común a *todas* las features que lo aplican) y además infla el build de features que ni la necesitan — cada módulo termina cargando dependencias que no usa, lo cual además reintroduce indirectamente el problema de acoplamiento que la modularización buscaba evitar. Lo correcto es declarar esa dependencia puntual en el `build.gradle.kts` propio de `:feature:cards`, después de aplicar el convention plugin base — exactamente como en el ejemplo mínimo de la sección 3, donde `:core:database` se agrega a mano en el módulo que lo necesita, no en el plugin.

## 6. Conexión con Timbax

Timbax hoy no tiene convention plugins porque no los necesita — es un único módulo `:shared`, así que no hay nada que centralizar entre módulos. Este archivo conecta directo con el anterior (`modularizacion_por_feature.md`): los convention plugins no son una herramienta independiente, son la solución al costo colateral de modularizar. Si algún día Timbax se modulariza (el escenario hipotético de `:core:scoring` + `:feature:chinchon`/`:feature:truco`/`:feature:generala` del archivo anterior), ese sería exactamente el momento de introducir un `build-logic` con un plugin `convention.kmp-game-feature` — porque ahí sí habría 3+ módulos repitiendo la misma config de targets y dependencias base de scoring.