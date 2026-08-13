# Fakes vs Mocks + Turbine

## 1. Qué es

**Fake** y **Mock** son dos formas distintas de reemplazar una dependencia real (una API, una DB) por algo controlable en un test. No son lo mismo aunque en conversación casual se usen como sinónimos:

- **Fake**: una implementación real y funcional de la interfaz, pero simplificada — sin red, sin DB real. Tiene comportamiento propio (guarda datos en una `MutableList` en memoria, por ejemplo).
- **Mock**: un objeto "hueco" que no tiene comportamiento propio — vos le programás explícitamente qué devolver ante cada llamada (`when(mock.getPlayers()).thenReturn(...)`), y además podés verificar que un método fue llamado N veces con ciertos argumentos.

**Turbine** es una librería (de Cash App) diseñada específicamente para testear `Flow`, `StateFlow` y `SharedFlow` de forma legible, sin tener que armar colectores manuales con `launch` + `toList()` + cancelación a mano.

## 2. El problema que resuelve

**Fake vs Mock resuelve:** cómo aislar la clase que estás testeando (un `UseCase`, un `ViewModel`) de sus dependencias reales, sin que el test dependa de una API que puede caerse, tardar, o requerir internet — algo inviable en un test unitario que corre cientos de veces por día en CI.

**Turbine resuelve:** testear un `Flow` a mano es incómodo y propenso a bugs de test — tenés que lanzar una coroutine para colectar, guardar los valores emitidos en una lista, y acordarte de cancelar la colección al final o el test queda colgado (un `Flow` frío nunca termina solo). Turbine te da una API (`.test { awaitItem() }`) que maneja todo eso por vos: colecta, espera el próximo valor con timeout automático, y cancela al salir del bloque.

## 3. Ejemplo mínimo comentado

### Fake — para el caso típico de UseCase/ViewModel

```kotlin
// Fake: implementación real y simple de la interfaz, vive en memoria
class FakePlayerRepository : PlayerRepository {

    private val players = mutableListOf<Player>()

    override fun getPlayers(): Flow<List<Player>> = flow { emit(players) }

    override suspend fun savePlayer(player: Player) {
        players.add(player) // comportamiento real, no un stub programado
    }
}

class SavePlayerUseCaseTest {

    private val fakeRepository = FakePlayerRepository()
    private val useCase = SavePlayerUseCase(fakeRepository)

    @Test
    fun `guarda el player y despues aparece en getPlayers`() = runTest {
        useCase(Player(id = "1", name = "Tiago", score = 0))

        val players = fakeRepository.getPlayers().first()
        assertEquals(1, players.size) // se verifica el RESULTADO, no que "se llamó a un método"
    }
}
```

### Mock — para el caso puntual donde SÍ hace falta

```kotlin
// Mock: no tiene comportamiento propio, se lo programás y se verifica interacción
class SendAnalyticsEventUseCaseTest {

    private val mockAnalytics = mock<AnalyticsClient>() // Mockative/MockMP

    @Test
    fun `dispara el evento de analytics con el nombre correcto`() = runTest {
        val useCase = SendAnalyticsEventUseCase(mockAnalytics)

        useCase("player_score_saved")

        // acá lo que importa es la INTERACCIÓN, no un resultado de dominio
        verify(mockAnalytics).logEvent(eq("player_score_saved"))
    }
}
```

### Turbine — testeando el `StateFlow` de un ViewModel

```kotlin
class PlayersViewModelTest {

    private val fakeRepository = FakePlayerRepository()
    private val viewModel = PlayersViewModel(GetPlayersUseCase(fakeRepository))

    @Test
    fun `al cargar, el state pasa por loading y despues muestra los players`() = runTest {
        viewModel.state.test {
            assertEquals(PlayersState(isLoading = false), awaitItem()) // estado inicial

            viewModel.onEvent(PlayersEvent.OnRefresh)

            assertEquals(true, awaitItem().isLoading)       // segundo emit: loading = true
            val loaded = awaitItem()
            assertEquals(false, loaded.isLoading)             // tercer emit: datos ya cargados
            assertTrue(loaded.players.isNotEmpty())

            cancelAndIgnoreRemainingEvents() // Turbine exige salir explícito del flow infinito
        }
    }
}
```

## 4. Matriz de criterio

**Usar Fake cuando:**
- Estás testeando `UseCase` o `ViewModel` y lo que te importa es el **resultado** (qué State quedó, qué devolvió el UseCase), no cómo se llegó ahí internamente.
- La dependencia tiene lógica simple de simular (una lista en memoria, un mapa) — es el caso normal para `Repository` en Clean Architecture.
- Vas a reusar la misma dependencia falsa en varios tests (un `FakeRepository` se reutiliza entre el test del `UseCase` y el del `ViewModel` que lo consume).

**Usar Mock cuando:**
- Lo que te importa es **verificar una interacción**, no un resultado de dominio — ejemplo típico: "¿se llamó a `analytics.logEvent()` exactamente una vez con este nombre de evento?". Ahí no hay "estado resultante" que chequear, solo la llamada en sí.
- La dependencia es difícil o directamente inviable de fakear con comportamiento real (un SDK de terceros con superficie enorme, tipo Firebase Auth) y solo necesitás simular un caso puntual de esa superficie.
- Necesitás simular explícitamente un error/excepción de una dependencia sin tener que programar lógica de fallo dentro de un Fake (`whenever(mock.getPlayers()).thenThrow(...)`).

**Trade-off real:** los Fakes dan tests más legibles y menos frágiles (no se rompen si cambiás detalles de implementación interna, porque el fake tiene comportamiento real), pero cuestan más armar al principio. Los Mocks son rápidos de armar para un caso puntual, pero acoplan el test a "cómo" se llama el método, no a "qué" pasa — si refactorizás la implementación sin cambiar el contrato, un test con mocks mal usados puede romperse sin razón real.

**Usar Turbine cuando:**
- Testeás cualquier `Flow`, `StateFlow` o `SharedFlow` — es la librería estándar KMP para esto, no hay razón real para colectar a mano salvo un caso muy específico no cubierto por su API.

## 5. Caso trampa

**"Voy a mockear el `Repository` para el test del `ViewModel`, así controlo exactamente qué devuelve."**

Esto compila y funciona, pero es la trampa más común de sobre-uso de Mock: terminás con un test que dice `whenever(mockRepository.getPlayers()).thenReturn(flowOf(listOf(playerFalso)))` — que es exactamente lo mismo que hace un `FakeRepository`, pero con más ceremonia (`whenever`, `thenReturn`, imports de librería de mocking) y menos legibilidad. Si tu `Repository` no tiene lógica de fallo que necesites simular puntualmente, el Fake es siempre la opción más limpia. Reservá el Mock para cuando realmente necesitás **verificar una llamada** o simular un caso imposible de modelar con un fake simple (ej: una excepción de red específica en el tercer llamado, no en el primero).

**Segunda trampa, con Turbine:** olvidarse de `cancelAndIgnoreRemainingEvents()` (o consumir exactamente la cantidad de items esperados) al final del bloque `.test { }`. Si tu `Flow` sigue emitiendo después de tu último `awaitItem()` y no cerrás el bloque, Turbine tira un error de "eventos no consumidos" — no es un detalle cosmético, es una verificación real de que tu test conoce **todas** las emisiones del flow, no solo las que te convenía chequear.

## 6. Conexión con Timbax

En Timbax, `FakePlayerRepository` es el caso de uso natural para testear tanto `SaveScoreUseCase` como el `ViewModel` que lo consume, sin tocar SQLDelight real. El Mock tendría sentido puntual si en algún momento se agrega un evento de Firebase Analytics al guardar un puntaje (`analytics.logEvent("score_saved")`) — ahí sí lo que importa es verificar que se llamó al SDK de GitLive, no simular un resultado de dominio. Turbine, por su parte, es directamente el método para testear el `state: StateFlow<PlayersState>` de cualquier ViewModel de Timbax que maneje loading/data/error, el patrón MVI que ya vimos en `06_presentation_mvi`.