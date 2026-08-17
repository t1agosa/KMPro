# Swift Export

## 1. Qué es

Swift Export es el mecanismo nuevo de Kotlin/Native para exponer código Kotlin directamente a Swift, sin pasar por el puente Objective-C que hoy usan `framework_xcframework_cinterop.md` e `interop_inverso_swift.md`. En vez de que el compilador traduzca las clases/funciones públicas del módulo `shared` a headers Objective-C (que Swift después interpreta con su propio sistema de bridging, generando nombres mangled y prefijos confusos), Swift Export genera directamente un módulo Swift nativo — imports limpios, sin intermediario. Es experimental desde Kotlin 2.2.20 (septiembre 2025) y fue promovido a **Alpha** en Kotlin 2.4.0 (2026), con soporte creciente: desde 2.4.0 exporta `suspend fun` como `async/await` nativo y `Flow` directamente como `AsyncSequence`, sin necesitar una librería puente como SKIE o KMP-NativeCoroutines para ese caso puntual. A la fecha, sigue en Alpha — JetBrains advierte explícitamente que todavía está incompleto y que hay que esperar cambios que rompan compatibilidad.

## 2. El problema que resuelve

Todo lo documentado en `interop_inverso_swift.md` sobre el puente Objective-C sigue siendo cierto y sigue funcionando — pero con fricción real: `suspend fun` se traduce a completion handlers poco idiomáticos, `Flow` no tiene traducción automática en absoluto sin una librería puente, y los nombres que le llegan a Swift a veces salen confusos porque pasaron por una capa de compatibilidad diseñada décadas atrás, sin pensar en generics modernos, sealed classes ni coroutines. Swift Export ataca el problema de raíz: en vez de traducir Kotlin → Objective-C → Swift (con pérdida de expresividad en cada salto), genera Swift directamente desde el compilador de Kotlin/Native — el mismo objetivo que ya resuelve el interop clásico, pero sin el intermediario que causa la mayoría de esa fricción.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — el mismo repository de interop_inverso_swift.md, sin cambios de código
class PlayerRepository {
    suspend fun getPlayers(): List<Player> { /* ... */ }
    fun observePlayers(): Flow<List<Player>> { /* ... */ }
}
```

```kotlin
// shared/build.gradle.kts — Swift Export es opt-in explícito, no es el comportamiento default
import org.jetbrains.kotlin.gradle.swiftexport.ExperimentalSwiftExportDsl

kotlin {
    iosArm64()
    iosSimulatorArm64()

    @OptIn(ExperimentalSwiftExportDsl::class)
    swiftExport {
        moduleName = "Shared"
        flattenPackage = "com.timbax.shared" // limpia el prefijo largo de paquete en el Swift generado
    }
}
```

```swift
// Swift, CON Swift Export (Kotlin 2.4+) — sin SKIE ni KMP-NativeCoroutines instalados
let players = try await playerRepository.getPlayers() // suspend fun -> async/await nativo

for await players in playerRepository.observePlayers() { // Flow -> AsyncSequence nativo
    // actualizar UI en cada nueva emisión
}
```

En Xcode además hay que reemplazar la Run Script phase que usa la tarea `embedAndSignAppleFrameworkForXcode` (interop clásico) por `embedSwiftExportForXcode` — son dos pipelines de build distintos, no coexisten transparentemente en el mismo target sin ese cambio.

## 4. Matriz de criterio

**Adoptar Swift Export hoy vs. seguir con el puente Objective-C:**
- Adoptar Swift Export cuando: es un proyecto nuevo o un prototipo, el equipo iOS valora la experiencia de desarrollo por sobre la estabilidad garantizada, y hay margen para absorber breaking changes en futuras versiones de Kotlin.
- Seguir con el puente Objective-C (`framework_xcframework_cinterop.md`/`interop_inverso_swift.md`) cuando: el proyecto ya está en producción, o simplemente no se puede permitir el riesgo de una herramienta que JetBrains mismo describe como incompleta y sujeta a cambios incompatibles.
- Trade-off: Swift Export da una API mucho más limpia e idiomática desde el día uno; el interop clásico da la certeza de que lo que funciona hoy sigue funcionando igual la semana que viene.

**`Flow` → `AsyncSequence` nativo (Swift Export 2.4+) vs. SKIE/KMP-NativeCoroutines:**
- Usar el soporte nativo de Swift Export cuando: el proyecto ya adoptó Swift Export en general (no tiene sentido sumarlo solo para este caso puntual si el resto del interop sigue siendo Objective-C) y corre en Kotlin 2.4.0+.
- Seguir con SKIE/KMP-NativeCoroutines cuando: el proyecto sigue con el puente Objective-C como base — ambas librerías siguen siendo la opción más madura y probada en producción para ese escenario, documentado en `interop_inverso_swift.md`.
- Trade-off: el soporte nativo elimina una dependencia externa, pero hereda toda la inestabilidad de que Swift Export en su conjunto todavía sea Alpha.

**`flattenPackage`/`moduleName` — cuándo configurarlos explícitamente:**
- Configurarlos cuando: el paquete Kotlin real (`com.timbax.shared.data.remote...`) es largo y generaría imports Swift incómodos — `flattenPackage` recorta ese prefijo del lado Swift sin tocar la estructura de paquetes real de Kotlin.
- Dejarlos por default cuando: el proyecto es chico y la jerarquía de paquetes ya es corta — el costo de no configurarlo es puramente cosmético (imports un poco más largos), no funcional.

**Exportar módulos separados (`export(project(":modulo"))`) vs. un solo módulo `shared` monolítico:**
- Usar exportación por módulo cuando: el proyecto ya está modularizado por feature (ver `modularizacion_por_feature.md`) y el equipo iOS se beneficia de importar solo lo que necesita, con nombres Swift independientes por módulo.
- Exportar todo como un único módulo cuando: el proyecto es chico o todavía no está modularizado — agregar la configuración de exportación por módulo antes de tener módulos reales que separar es complejidad sin beneficio.

## 5. Caso trampa

Asumir que, porque `suspend fun` y `Flow` ya se exportan bien desde Kotlin 2.4.0, cualquier otra construcción de Kotlin se exporta con la misma cobertura — en particular, las `sealed class`/`sealed interface` que este mismo repo usa en cada `Contract` de MVI (`State`, `Event`, `Effect`, ver `contract_state_event_effect.md`):

```kotlin
// commonMain — exactamente el patrón que este repo documenta para MVI
sealed interface PlayersEffect {
    data class NavigateToPlayerDetail(val playerId: String) : PlayersEffect
    data class ShowSnackbar(val message: String) : PlayersEffect
}
```

El soporte de Swift Export para `suspend fun` y `Flow` llegó en un release (Kotlin 2.4.0); el soporte para exportar `sealed class`/`sealed interface` de forma completa llegó en releases posteriores (2.4.20 en adelante). La trampa es leer "Swift Export ya soporta coroutines" y asumir que eso implica cobertura pareja de todo lo demás — dos features de la misma herramienta pueden madurar en momentos completamente distintos. Un proyecto que fija su versión de Kotlin en 2.4.0 y da por sentado que puede exportar directamente sus `Contract` de MVI (que en este repo son, en su enorme mayoría, `sealed interface`) puede encontrarse con que justo la construcción que más usa es la que menos cobertura tiene en esa versión puntual. La corrección: antes de decidir exportar algo vía Swift Export, verificar contra el changelog de la versión exacta de Kotlin del proyecto qué construcciones soporta — no contra la fecha en que "salió Swift Export" en general.

## 6. Conexión con Timbax

Timbax hoy no tiene un target iOS en producción, así que Swift Export no es todavía una decisión que el proyecto necesite tomar — pero es la pieza que faltaba en el mapa de `10_ios_interop` para cuando esa conversación exista. La recomendación de este archivo, consistente con el principio ya documentado en `14_criterio_y_decisiones` de que "no implementar algo es una decisión válida": si Timbax sumara un target iOS hoy, el camino recomendado seguiría siendo el puente Objective-C ya documentado en `framework_xcframework_cinterop.md` e `interop_inverso_swift.md` — probado, estable, sin sorpresas de versión — dejando Swift Export como algo para prototipar en paralelo y adoptar recién cuando alcance una madurez que un proyecto real (no un playground) pueda sostener. Es la misma lógica que ya se aplicó con Appdome/Play Integrity en el módulo 16: conocer la herramienta y su criterio de adopción es tan parte del trabajo senior como saber usarla.