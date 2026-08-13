# Kotlin/Native

## 1. Qué es

El compilador de Kotlin que compila tu código directamente a binario nativo, sin pasar por una máquina virtual. Usa LLVM por debajo para generar código máquina específico de la plataforma destino (ARM64 para iOS, x86_64 para simulador, etc). Es el motor que hace posible que el módulo `commonMain` de Timbax termine corriendo en un iPhone sin que exista una JVM en el medio.

Importante: no es "un compilador más de Kotlin" — es un compilador *distinto* al que usás para Android. Tu mismo código fuente en `commonMain` se compila dos veces, de dos formas totalmente diferentes según la plataforma:

- **Android** → Kotlin/JVM → bytecode JVM (como cualquier app Android tradicional).
- **iOS** → Kotlin/Native → binario nativo ARM64/x86_64 (como si hubieras escrito Swift u Objective-C directamente).

## 2. El problema que resuelve

iOS no tiene máquina virtual de Java disponible — no hay forma de correr bytecode JVM en un iPhone. Si Kotlin solo pudiera compilar a JVM, KMP terminaría ahí: "Kotlin Multiplatform" sería en realidad "Kotlin Android + algo más liviano en otros lados", nunca código de dominio/lógica compartido de verdad corriendo nativamente en iOS.

Kotlin/Native resuelve exactamente ese problema: te permite escribir la lógica una sola vez en Kotlin puro (tus `UseCase`, tu `Repository`, tu `ViewModel`) y que termine siendo binario nativo en iOS, con el mismo nivel de integración que tendría código escrito directamente en Swift.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — este código se compila DOS VECES, con DOS compiladores distintos
class SaveScoreUseCase(private val repository: PlayerRepository) {
    suspend operator fun invoke(playerId: String, score: Int) {
        repository.updateScore(playerId, score)
    }
}
```

```
Compilación para Android → Kotlin/JVM → bytecode .class/.dex → corre en la JVM/ART de Android
Compilación para iOS     → Kotlin/Native → binario ARM64/x86_64 → corre nativo, sin VM
```

No hay ninguna línea de código distinta acá — el mismo `SaveScoreUseCase.kt` es el input de ambos compiladores. La diferencia está enteramente en el paso de compilación, no en tu código fuente.

## 4. Matriz de criterio

| Usar cuando | NO aplica cuando | Trade-off real |
|---|---|---|
| Necesitás que código Kotlin puro (`commonMain`) corra en iOS | Estás pensando en "optimizar" código para Kotlin/Native específicamente — no es una decisión que tomás vos, es automática por target | Los tiempos de compilación para iOS suelen ser más lentos que para Android (LLVM + linking nativo vs bytecode JVM) |
| Vas a llamar APIs nativas de iOS desde Kotlin (vía `expect/actual` + cinterop) | Tu lógica depende de una librería JVM-only (ej: Retrofit) — esa librería directamente no compila para Kotlin/Native, necesitás una alternativa KMP (Ktor) | Debugging de crashes nativos en iOS requiere herramientas distintas (crash logs de Xcode/LLDB) a las que usás para Android |

## 5. Caso trampa

**Pregunta:** *"Si mi `ViewModel` y mis `UseCase` están en `commonMain` y ya funcionan perfecto en Android, ¿por qué no compilan directo para iOS sin tocar nada?"*

Respuesta ingenua: "Deberían compilar, es el mismo código Kotlin."

Respuesta correcta: el código en sí compila, pero **cualquier dependencia que uses en `commonMain` también tiene que soportar Kotlin/Native**, no solo Kotlin/JVM. Si en algún punto de tu cadena de dependencias (directa o transitiva) hay una librería JVM-only — el ejemplo clásico es usar Retrofit en vez de Ktor, o una librería de logging tipo Timber en vez de Kermit — el build para iOS se rompe en tiempo de compilación, aunque tu código de Timbax esté perfecto. La trampa es asumir que "es Kotlin, así que compila en todos lados": compila en todos lados *si y solo si* toda la cadena de dependencias también es KMP.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, todo lo que vive en `domain` (tus `UseCase`, tus modelos, tus interfaces de `Repository`) y buena parte de `data`/`presentation` está en `commonMain`, precisamente para aprovechar Kotlin/Native y que corra en iOS sin reescribir nada. Es la razón de fondo por la que elegiste Ktor sobre Retrofit y Kermit sobre Timber en el machete original: no es preferencia estética, es que Retrofit y Timber son Kotlin/JVM-only y directamente no compilan bajo Kotlin/Native, así que romperían el build de iOS apenas los importaras en `commonMain`.