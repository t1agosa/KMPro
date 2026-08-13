# Estrategia y prioridades de testing

## 1. Qué es

La estrategia de testing es la decisión de **qué se testea, en qué orden, y en qué momento del ciclo de desarrollo** — no es "escribir tests", es decidir dónde conviene invertir el tiempo limitado que tenés para testear, porque nadie testea el 100% de una app real con el mismo nivel de detalle en todas las capas.

En KMP con Clean Architecture, la prioridad es directa porque la arquitectura ya te la sugiere: se testea más y primero donde vive la lógica pura (domain), y cada vez menos a medida que te acercás a UI/plataforma, porque ahí el costo de testear sube y el valor relativo baja (Compose Preview + testing manual cubren gran parte de lo visual).

## 2. El problema que resuelve

La pregunta real es: **¿cuándo testea un dev en un trabajo real?**

La respuesta corta es: **casi nunca es TDD puro (test primero, código después) en el día a día de una feature nueva**, aunque en teoría/entrevistas suene como "la forma correcta". Lo que pasa en la práctica en la mayoría de equipos:

**Orden real, capa por capa:**

1. **Domain primero, siempre** — se define el `Model` y la firma del `UseCase` (`operator fun invoke(...)`) antes que nada, porque es el contrato que todo lo demás va a consumir.
2. **¿Test antes o después del UseCase?** Depende de la complejidad de la regla de negocio:
    - Si la regla es simple (un mapeo, una validación de un campo) → se escribe el código primero, el test se agrega inmediatamente después, en el mismo bloque de trabajo, antes de pasar a la siguiente clase. No hay tiempo real de "codear sin red de seguridad" que se recupere después — se recupera ahora o no se recupera.
    - Si la regla es compleja o tiene casos borde no triviales (ej: cálculo de puntaje con bonificaciones, reglas de un juego de cartas tipo Chinchón) → ahí sí conviene escribir 2-3 tests primero (los casos borde que ya sabés que existen) porque te obliga a pensar la función antes de escribirla, y es más barato encontrar el bug en el test que en el `UseCase` ya integrado con el resto.
3. **Data (Repository + mappers)** → se testea después de tener el `UseCase` funcionando, normalmente con un `FakeRepository` que ya usaste para testear el `UseCase` en el paso 2 (mismo fake, doble uso). El mapper (`Dto.toDomain()`) se testea aparte y rápido porque es una función pura sin dependencias.
4. **Presentation (ViewModel/reducer)** → se testea al final de la capa de lógica, antes de tocar UI. El patrón real es: escribís el `Contract` (State/Event/Effect) completo primero (sin tests, es solo definición de tipos), después el `ViewModel`, y ahí sí los tests son casi obligatorios porque el `ViewModel` es donde más bugs de estado se filtran a producción (combinaciones de loading/error/data mal manejadas).
5. **UI (Compose)** → normalmente NO se testea con tests automatizados en el día a día, salvo en equipos grandes con QA dedicado. Se valida con Preview + testing manual. Los `UiTest` con `ComposeTestRule` existen pero son la última prioridad, no la primera.

**El momento clave:** el punto de "no retorno" real en un equipo profesional no es "escribo el código y algún día testeo" — es **"no se abre Pull Request sin tests de domain y presentation"**. Ese es el gate real. Nadie te obliga a hacer TDD estricto, pero sí te van a rechazar un PR de un `UseCase` con lógica no trivial sin al menos 2-3 tests cubriendo los casos principales y un caso borde.

**¿Y con IA en el medio?** Acá cambia respecto a hace unos años. El flujo real hoy tiene sentido así:

1. Describís la feature completa a la IA (rol ARQUITECTO): domain, contract MVI, etc.
2. La IA genera `Model` + `UseCase` + `Repository` interface.
3. **Inmediatamente después de aceptar ese código — mismo bloque de trabajo, no "después"** — le pedís los tests de ese `UseCase` a la IA (rol TESTING + QA). No hay separación temporal real entre "código" y "test" cuando la IA genera ambos rápido: el costo de escribir el test bajó tanto que la excusa de "lo hago después" ya no aplica. La dupla código+test es la unidad mínima de trabajo terminado, no dos tareas separadas.
4. Se repite por capa: domain completo (código+test) → data (código+test) → presentation (código+test) → recién ahí UI.

Esto no es TDD (no se escribe el test antes), pero tampoco es "código y algún día testeo" — es **"código y test como una sola entrega"**, el patrón más realista para un dev productivo con IA.

## 3. Ejemplo mínimo comentado

```kotlin
// 1. Domain: se define primero, sin test todavía (es solo el contrato)
class SaveScoreUseCase(private val repository: PlayerRepository) {
    operator fun invoke(playerId: String, currentScore: Int, points: Int): Result<Int> {
        val newScore = currentScore + points
        if (newScore < 0) {
            return Result.Error(IllegalArgumentException("El score no puede ser negativo"))
        }
        return Result.Success(newScore)
    }
}
```

```kotlin
// 2. Test: se escribe INMEDIATAMENTE después, misma sesión de trabajo,
// antes de pasar a la siguiente clase — no queda pendiente para "después"
class SaveScoreUseCaseTest {

    private val fakeRepository = FakePlayerRepository()
    private val useCase = SaveScoreUseCase(fakeRepository)

    @Test
    fun `suma puntos correctamente al score actual`() {
        val result = useCase(playerId = "1", currentScore = 50, points = 10)

        assertTrue(result is Result.Success)
        assertEquals(60, (result as Result.Success).data)
    }

    @Test
    fun `rechaza el resultado si el score queda negativo`() {
        // este es el caso borde que justifica el test —
        // sin él, un bug de este tipo se descubre en producción, no en CI
        val result = useCase(playerId = "1", currentScore = 5, points = -20)

        assertTrue(result is Result.Error)
    }
}
```

El punto no es el código en sí — es que **el segundo bloque se escribe en el mismo momento que el primero**, no en una sesión de "testing" separada una semana después.

## 4. Matriz de criterio

**Testear ANTES de escribir el código (TDD real) cuando:**
- La regla de negocio tiene casos borde no obvios que necesitás pensar antes de codear (ej: cálculo de puntaje con reglas especiales de un juego de cartas).
- Estás corrigiendo un bug reportado: primero escribís el test que reproduce el bug (falla), después arreglás el código (el test pasa). Esto es TDD aplicado de forma quirúrgica, no como filosofía general.

**Testear INMEDIATAMENTE después (patrón más común) cuando:**
- Es un `UseCase`, mapper, o reducer con lógica directa — el flujo natural con o sin IA: código, después test, mismo bloque de trabajo, antes de pasar a la siguiente clase.
- Estás generando código con IA: pedís el código y el test en la misma interacción o la siguiente inmediata, no como tareas separadas en el tiempo.

**Testear DESPUÉS, en un bloque separado (revisión de cobertura) cuando:**
- Cerrás un package completo (ej: terminaste toda `presentation`) y hacés una pasada de "¿qué me quedó sin cubrir?" antes de abrir el PR — esto es un checkpoint de calidad, no la primera línea de defensa.

**NO testear (o testear mínimo) cuando:**
- Es un `Composable` puramente visual sin lógica condicional compleja — Preview + revisión manual alcanza.
- Es código de plataforma (`expect/actual`) que envuelve una API nativa ya probada por el SO — testear ahí es testear el SDK de Apple/Google, no tu lógica.
- Es un DTO o modelo de datos sin comportamiento, solo propiedades.

**Trade-off real:** más tests = más confianza para refactorizar sin miedo, pero también más tiempo de mantenimiento cuando cambia la firma de algo. Por eso domain (que cambia poco una vez estable) se testea a fondo, y UI (que cambia todo el tiempo por diseño) se testea poco.

## 5. Caso trampa

**"Ya generé el `UseCase` con la IA, se ve simple, no hace falta testearlo ahora — lo hago cuando termine toda la feature."**

Esta es la trampa más común y la que arruina el flujo real: "lo hago después" casi nunca pasa, porque cuando terminás la feature completa ya tenés 8 clases nuevas y "escribir los tests" se convirtió en una tarea gigante y aburrida que se posterga otra vez, o se hace apurado justo antes del PR con cobertura superficial.

La forma correcta: el test del `UseCase` se pide/escribe en el mismo momento en que aceptás ese `UseCase`, aunque "se vea simple". Un `UseCase` que hoy es simple mañana tiene un caso borde nuevo que alguien va a agregar sin volver a revisar si rompió algo — y ahí es donde el test ya escrito te salva, no el test que "ibas a escribir después".

**Segunda trampa relacionada:** pensar que como la IA generó el código, el código está "probado" solo por haber sido generado por IA. No — la IA generando el `UseCase` y la IA generando el test son dos pasos separados con la misma probabilidad de error humano+IA en el medio (vos validando el código, no la IA validándose a sí misma). El test es la única verificación real, generado o no por IA.

## 6. Conexión con Timbax

En Timbax, el candidato más claro para el patrón "test justo después del código" es cualquier `UseCase` que calcule puntaje — es exactamente el tipo de lógica con casos borde (puntajes negativos, bonificaciones, reglas específicas del juego) donde escribir el test en el momento evita que un bug de cálculo llegue a producción, que en una app de scoring es el peor lugar posible para tener un bug silencioso (el usuario confía en que el número que ve es correcto, y no hay forma visual de detectar que está mal).