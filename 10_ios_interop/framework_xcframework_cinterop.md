# Framework / XCFramework / Cinterop

## 1. Qué es

Tres piezas relacionadas que resuelven el mismo problema desde ángulos distintos: **cómo el mundo Kotlin y el mundo Swift/Objective-C se comunican en ambas direcciones**.

- **Framework**: el formato binario que Xcode entiende y puede embeber en un proyecto iOS. Tu módulo `shared` de KMP no se consume directamente desde Swift — se compila a un `.framework`, que es la unidad de distribución estándar de Apple para código nativo (lo mismo que produciría un proyecto Swift/Objective-C compilado como librería).
- **XCFramework**: un contenedor que agrupa **varios** `.framework` — uno por cada arquitectura/target (simulador ARM64, dispositivo físico ARM64, Intel, etc). Xcode elige automáticamente el binario correcto según dónde estés compilando. Es necesario porque un solo `.framework` no puede contener simultáneamente binarios para simulador y dispositivo físico sin conflictos.
- **Cinterop**: la herramienta que genera *bindings* Kotlin para librerías C/Objective-C ya existentes. Es lo que permite que tu código en `iosMain` pueda llamar directamente APIs de UIKit, Core Location, Core Foundation, etc., como si fueran clases Kotlin normales.

## 2. El problema que resuelve

Sin estas tres piezas, KMP no podría integrarse de verdad con un proyecto iOS real:

- Sin **Framework/XCFramework**, no habría forma de que Xcode "entienda" el binario que produce Kotlin/Native — necesitás el formato que la toolchain de Apple sabe consumir, con headers y metadata compatibles.
- Sin **Cinterop**, tu código Kotlin en `iosMain` quedaría aislado del ecosistema nativo de iOS — no podrías llamar `UIDevice.currentDevice`, `UIKitView`, ni ninguna API del SDK de Apple, porque esas APIs están escritas en Objective-C/C, no en Kotlin.

En conjunto resuelven el problema de interoperabilidad en las dos direcciones: Cinterop te deja llamar código nativo *desde* Kotlin, y Framework/XCFramework te deja exponer tu código Kotlin *hacia* Swift.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — declaración expect, sin implementación
expect fun getDeviceId(): String
```

```kotlin
// iosMain — la implementación actual llama directo a UIKit vía cinterop
// "platform.UIKit" es un binding generado automáticamente por cinterop
// a partir de los headers de Objective-C del SDK de iOS
actual fun getDeviceId(): String =
    platform.UIKit.UIDevice.currentDevice.identifierForVendor?.UUIDString ?: ""
```

```kotlin
// build.gradle.kts (módulo shared) — configuración típica de framework
kotlin {
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach {
        it.binaries.framework {
            baseName = "shared" // nombre del framework que va a importar Swift
        }
    }
}
```

```swift
// del lado Swift, una vez embebido el XCFramework en Xcode:
import shared

let deviceId = SharedKt.getDeviceId() // tu función Kotlin, consumida como si fuera nativa
```

## 4. Matriz de criterio

| Usar cuando | NO aplica cuando | Trade-off real |
|---|---|---|
| Necesitás distribuir el módulo `shared` a un proyecto Xcode que corre en simulador **y** dispositivo físico | Solo estás compilando para un único target fijo (raro en la práctica, pero ahí un `.framework` simple alcanzaría) | Generar un XCFramework agrega un paso extra al build (`assembleXCFramework` en vez de un simple `framework`), y el tamaño del artefacto crece por incluir varios binarios |
| Necesitás llamar una API de UIKit/Core Foundation/cualquier librería Objective-C desde `iosMain` | Estás en `commonMain` (cinterop solo aplica dentro de código específico de plataforma iOS, nunca en código compartido) | Los bindings generados por cinterop pueden ser verbosos o poco idiomáticos comparados con Swift nativo — a veces necesitás wrappers propios en `iosMain` para limpiar la API antes de exponerla |
| El equipo iOS quiere manejar el framework como una dependencia versionada, no copiada a mano | El proyecto es un prototipo chico de una sola persona — ahí embeber el framework directo en Xcode sin CocoaPods/SPM es más simple | CocoaPods/SPM agregan configuración extra en el `build.gradle.kts` (plugin `cocoapods {}` o config de SPM) que hay que mantener sincronizada con el resto del build KMP |

## 5. Caso trampa

**Pregunta:** *"El equipo iOS me pide 'el framework' para probar la app en su simulador. ¿Les paso el `.framework` que me genera Android Studio en `build/bin/iosSimulatorArm64/`?"*

Respuesta ingenua: "Sí, ahí está el framework, se lo mando."

Respuesta correcta: ese `.framework` generado para `iosSimulatorArm64` **solo sirve para simulador Apple Silicon** — si alguien del equipo iOS lo prueba en un dispositivo físico (`iosArm64`) o en un Mac Intel (`iosX64`), no va a andar, porque cada target genera un binario distinto y no intercambiable. La trampa es asumir que "un framework es un framework" — en la práctica, para distribuir algo que funcione en cualquier combinación de simulador/dispositivo, necesitás generar el **XCFramework** (que empaqueta los tres binarios juntos) con la tarea de Gradle correspondiente, no copiar a mano la carpeta de un solo target.

## 6. Conexión con arquitectura real (Timbax)

Si mañana el equipo de Timbax decide construir la versión iOS, todo el trabajo hecho en `domain`, `data` y buena parte de `presentation` (que ya vive en `commonMain` por diseño de Clean Architecture) se compilaría directo a un XCFramework consumible desde Xcode, sin reescribir la lógica de negocio. Los únicos puntos donde entraría Cinterop serían las funciones `expect/actual` que ya existen en el proyecto — como el acceso a plataforma que hoy resuelve `expect fun getPlatformName()` — donde el lado `iosMain` necesitaría llamar APIs reales de UIKit en vez de la implementación simulada que pueda existir hoy solo pensando en Android.