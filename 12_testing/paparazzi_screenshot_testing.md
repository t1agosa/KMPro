# Paparazzi (Screenshot Testing)

## 1. Qué es

Paparazzi es una librería de Cash App para **screenshot testing** (también llamado snapshot testing): renderiza un composable (o una `View` clásica) en la JVM — sin emulador ni dispositivo físico — usando `LayoutLib`, la misma librería privada que usa Android Studio para renderizar los Compose Previews. El resultado se guarda como una imagen "golden" (la referencia correcta), y en cada corrida posterior Paparazzi vuelve a renderizar el mismo composable y lo compara píxel a píxel contra esa imagen — si hay una diferencia visual real, el test falla. Es la capa de testing que responde una pregunta distinta a todo lo demás en `12_testing`: no "¿la lógica es correcta?" (`fakes_vs_mocks_turbine.md`) ni "¿la interacción funciona?" (`espresso_testing_ui.md`), sino **"¿se sigue viendo igual?"**.

## 2. El problema que resuelve

Una regresión visual (un padding que cambió, un color que no se aplicó, un texto que ahora se corta) casi nunca rompe un test de lógica — el `ViewModel` puede estar devolviendo el `State` perfectamente correcto, y aun así la UI verse mal. Tampoco lo detecta necesariamente Espresso/Compose Testing: `assertIsDisplayed()` confirma que el elemento está en pantalla, no que tenga el color o el espaciado correctos. El QA manual atrapa algunas de estas regresiones, pero no escala — nadie revisa pixel a pixel cada pantalla en cada PR. Paparazzi resuelve esto corriendo tan rápido como un test unitario (no hay emulador que bootear), lo que lo hace viable correrlo en **cada Pull Request**, atrapando exactamente la categoría de bug que el resto de la pirámide de testing no cubre.

## 3. Ejemplo mínimo comentado

```kotlin
// build.gradle.kts del módulo — Paparazzi exige que el módulo sea com.android.library,
// NUNCA com.android.application (los módulos "application" generan recursos en bytecode
// que LayoutLib no puede leer — es una limitación real de la herramienta, no configuración)
plugins {
    id("com.android.library")
    id("app.cash.paparazzi")
}
```

```kotlin
class PlayerCardScreenshotTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_5,
        theme = "android:Theme.Material.Light.NoActionBar"
    )

    @Test
    fun playerCard_withHighScore() {
        paparazzi.snapshot {
            TimbaxTheme {
                PlayerCard(player = Player(id = "1", name = "Tiago", score = 42))
            }
        }
    }
}
```

`./gradlew recordPaparazzi` genera la imagen golden la primera vez (se commitea al repo junto con el código). `./gradlew verifyPaparazzi` vuelve a renderizar el mismo composable y compara contra esa imagen guardada — cualquier diferencia visual real por encima de la tolerancia configurada rompe el build.

## 4. Matriz de criterio

**Paparazzi vs. Compose Preview manual:**
- Usar Paparazzi cuando: querés una red de seguridad automática que corra en CI y bloquee un PR ante una regresión visual real — el Preview no corre solo, depende de que un humano lo mire.
- Seguir usando Preview cuando: estás iterando activamente el diseño de un composable — Preview es para el momento de construir, Paparazzi es para el momento de proteger lo ya construido.
- Trade-off: Paparazzi agrega mantenimiento real (actualizar goldens cuando un cambio visual es intencional); Preview no tiene ningún costo de mantenimiento porque no persiste nada.

**`recordPaparazzi` vs. `verifyPaparazzi`:**
- Correr `record` cuando: es la primera vez que se testea ese composable, o cuando un cambio visual fue **intencional** y hay que actualizar la referencia — siempre revisando visualmente el diff antes de commitear la nueva imagen golden (ver Caso trampa).
- Correr `verify` cuando: es el paso que corre en CI en cada PR — nunca debería correr `record` en CI de forma automática, porque eso "aceptaría" cualquier regresión visual como si fuera la nueva normalidad sin revisión humana.
- Trade-off: ninguno real — son dos comandos con propósitos claramente distintos, el error está en confundir cuál corre dónde.

**`com.android.library` vs. `com.android.application` para los composables testeados:**
- Mantener los composables reusables (cards, botones custom, componentes de diseño) en un módulo `library` cuando: querés poder testearlos con Paparazzi — es un argumento más a favor de la modularización por feature (`modularizacion_por_feature.md`), más allá del beneficio de build incremental ya documentado ahí.
- Aceptar la limitación cuando: el proyecto es chico y no está modularizado — en ese caso, Paparazzi simplemente no puede testear composables que viven directo en el módulo `app`.

**Paparazzi vs. Roborazzi:**
- Usar Paparazzi cuando: la prioridad es velocidad — renderiza vía `LayoutLib` sin correr el framework de Android real, es la opción más rápida del ecosistema.
- Usar Roborazzi cuando: el composable interactúa con partes reales del framework de Android (a través de Robolectric) que `LayoutLib` no simula bien — Roborazzi corre sobre un entorno Android más completo, a costa de ser más lento.
- Trade-off: Paparazzi gana en velocidad y simplicidad; Roborazzi gana en fidelidad cuando el componente depende de comportamiento real de Android que LayoutLib no reproduce (los mismos límites que a veces se ven en el Preview de Android Studio).

## 5. Caso trampa

Testear con Paparazzi un composable que carga una imagen real de forma asincrónica (una URL de avatar vía Coil/`AsyncImage`), sin fakear esa carga:

```kotlin
// ❌ trampa: PlayerCard carga el avatar desde una URL real
@Composable
fun PlayerCard(player: Player) {
    Row {
        AsyncImage(model = player.avatarUrl, contentDescription = null) // 🚩 carga async real
        Text(player.name)
    }
}

@Test
fun playerCard_snapshot() {
    paparazzi.snapshot {
        TimbaxTheme { PlayerCard(player = playerConAvatarReal) }
    }
}
```

Paparazzi renderiza un frame estático vía `LayoutLib` — no es un emulador con un ciclo de vida real donde esperar de forma determinística a que una carga de imagen de red termine. El resultado es un test flaky: a veces el snapshot sale con la imagen cargada, a veces con el placeholder, a veces en blanco, dependiendo de timing que el test no controla. Esto es peor que no tener el test: un screenshot test flaky erosiona la confianza en toda la suite, y el equipo empieza a ignorar fallos de CI relacionados asumiendo que "es cosa de Paparazzi otra vez". La corrección es la misma idea ya documentada en `fakes_vs_mocks_turbine.md` aplicada a lo visual: fakear la dependencia — inyectar un `ImageLoader` de test con un drawable local fijo, no una URL real — para que el snapshot sea 100% determinístico, sin red de por medio.

## 6. Conexión con Timbax

En Timbax, los candidatos naturales para Paparazzi son los componentes reusables ya documentados en `09_ui_compose` — `GameScoreText`, `PlayerCard`, los componentes de `material3_componentes_comunes.md` — no pantallas completas, donde la superficie de cosas que pueden variar (animaciones en curso, datos dinámicos) hace que la tolerancia sea más difícil de calibrar sin falsos positivos. Si Timbax modulariza por feature en el futuro (`modularizacion_por_feature.md`), esos componentes reusables quedarían naturalmente en un módulo `library`, habilitando Paparazzi sin fricción extra. Encaja como la capa más alta de la pirámide de testing que ya describe `estrategia_y_prioridades.md`: domain y presentation primero, UI casi nunca — y cuando sí se invierte en UI, Paparazzi cubre exactamente lo que ni un test de `ViewModel` ni un test de Espresso pueden ver: que la pantalla se siga viendo como se supone que se vea.