# sourceset_hierarchy.md

## 1. Qué es

Es la organización jerárquica de los `sourceSets` de Gradle en un proyecto KMP: no son solo `commonMain`, `androidMain` e `iosMain` como tres compartimentos aislados, sino un árbol donde podés crear sourceSets **intermedios** que agrupan varios targets emparentados (ej: `appleMain` compartido entre `iosMain`, `macosMain` y `watchosMain`; o `jvmCommonMain` compartido entre `androidMain` y `desktopMain`). Cada sourceSet hereda automáticamente todo lo que está definido en sus padres en la jerarquía.

```kotlin
// build.gradle.kts (módulo shared)
kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    macosX64()

    sourceSets {
        val commonMain by getting
        val appleMain by creating {
            dependsOn(commonMain)
        }
        val iosX64Main by getting { dependsOn(appleMain) }
        val iosArm64Main by getting { dependsOn(appleMain) }
        val iosSimulatorArm64Main by getting { dependsOn(appleMain) }
        val macosX64Main by getting { dependsOn(appleMain) }
    }
}
```

## 2. El problema que resuelve

Sin sourceSets intermedios, un `actual` que aplica igual a iOS y macOS (por ejemplo, algo implementado con Foundation/UIKit-adjacent APIs que existen en todo el ecosistema Apple) tendría que duplicarse literalmente en `iosMain` y en `macosMain` — mismo código, copiado dos veces, con el riesgo de que diverjan con el tiempo si alguien actualiza uno y se olvida el otro. La jerarquía de sourceSets resuelve esto dejando declarar el `actual` **una sola vez** en el nivel intermedio (`appleMain`) que ambos targets heredan, mientras que las partes que sí difieren entre iOS y macOS específicamente (si las hay) siguen yendo en sus sourceSets hoja respectivos. Es la misma lógica de "no te repitas" aplicada a la estructura de compilación, no solo al código.

## 3. Ejemplo mínimo comentado

```kotlin
// appleMain — un solo actual para toda la familia Apple
actual fun getSecureStorageKey(): String =
    platform.Security.kSecClassGenericPassword.toString()
```

```kotlin
// iosMain — solo lo que es específico de iOS, no de toda la familia Apple
actual fun getHapticFeedbackStyle(): String = "UIImpactFeedbackGenerator"
```

```kotlin
// macosMain — lo específico de macOS, no comparte con iOS
actual fun getHapticFeedbackStyle(): String = "NSHapticFeedbackManager"
```

Nota: `getSecureStorageKey()` vive una sola vez en `appleMain` y la heredan tanto `iosMain` como `macosMain` sin declarar nada extra; `getHapticFeedbackStyle()` sí necesita un `actual` distinto en cada uno porque el comportamiento real difiere.

## 4. Matriz de criterio

**Usar un sourceSet intermedio cuando:**
- Dos o más targets de la misma familia (Apple, JVM, Native en general) comparten exactamente el mismo `actual`, y ese código usa APIs disponibles en todos esos targets sin excepción.
- El proyecto ya declara más de 2-3 targets emparentados (si solo tenés `androidTarget()` + `iosX64()`/`iosArm64()`/`iosSimulatorArm64()`, KMP ya te da `iosMain` como sourceSet intermedio automático entre los tres — no hace falta crearlo a mano).

**NO usar un sourceSet intermedio cuando:**
- Solo tenés un target por familia (ej: un solo target Android, sin variantes) — no hay nada que agrupar, sería una capa vacía.
- El comportamiento realmente difiere entre los targets "hermanos" — forzar un `actual` común en el sourceSet intermedio cuando en realidad iOS y macOS necesitan lógica distinta te obliga a meter un `if`/`when` de plataforma *dentro* del `actual` compartido, lo cual reintroduce el problema que `expect/actual` existe para evitar (ver `expect_actual.md`).

**Trade-off real:** los sourceSets intermedios reducen duplicación pero agregan una capa más a la jerarquía que hay que entender al leer el proyecto — en un repo chico con pocos targets, declarar `appleMain` manualmente antes de necesitarlo realmente es complejidad prematura; conviene agregarlo recién cuando aparece la primera duplicación real entre dos targets emparentados, no antes.

## 5. Caso trampa

El proyecto tiene `iosMain` (automático, agrupa `iosX64Main`/`iosArm64Main`/`iosSimulatorArm64Main`) y funciona bien. Se agrega `macosX64()` como nuevo target para una versión de escritorio en Mac. La tentación es poner el `actual` que ya existía en `iosMain` directamente también ahí, duplicado — "totalmente similar, es Apple igual". Pero `iosMain` en KMP es un sourceSet específico de iOS, no de "todo lo Apple" — no existe automáticamente un padre común entre `iosMain` y `macosMain` a menos que lo crees vos mismo (`appleMain`, como en el ejemplo). Si simplemente duplicás el código en `macosMain` en vez de crear el sourceSet intermedio, funciona hoy, pero perdiste la garantía de que ambas copias se mantengan sincronizadas — es exactamente el problema que la jerarquía de sourceSets está pensada para eliminar, y es fácil no darse cuenta porque "compila igual" de las dos formas.

## 6. Conexión con arquitectura real

Timbax hoy corre en Android e iOS — con la jerarquía automática que KMP da por default (`commonMain` → `androidMain` / `iosMain`, y este último ya agrupando los tres targets de iOS internamente) alcanza sin necesidad de sourceSets intermedios manuales. El día que se sume un target Desktop (JVM) o una segunda plataforma Apple, ahí es cuando conviene revisar si algún `actual` que hoy vive solo en `iosMain` debería subir a un `appleMain` creado a mano, siguiendo el mismo criterio de "duplicación real detectada, no anticipada" que se aplica en general antes de agregar abstracción.