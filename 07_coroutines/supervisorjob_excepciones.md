# 07_coroutines / `supervisorjob_excepciones.md`

## 1. Mapa del flujo

```mermaid
flowchart TD
    A["CoroutineScope(Job() + ...)"] -->|"launch hijo 1"| B[Falla con excepción]
    A -->|"launch hijo 2"| C[Sigue corriendo]
    B -.->|"Job normal: se propaga hacia arriba"| D[Cancela TODO el scope]
    D -.-> C

    E["CoroutineScope(SupervisorJob() + ...)"] -->|"launch hijo 1"| F[Falla con excepción]
    E -->|"launch hijo 2"| G[Sigue corriendo, no se entera]
    F -->|"aislado, no se propaga"| H[Solo esa coroutine termina]

    I["scope.cancel()"] -->|"lanza CancellationException"| J{"¿El catch la relanza?"}
    J -->|"No, la traga"| K[Cancelación rota: la coroutine sigue viva]
    J -->|"Sí, la relanza"| L[Cancelación cooperativa correcta]
```

Este diagrama tiene dos partes independientes que responden a la misma pregunta de fondo — "¿qué pasa cuando algo sale mal dentro de una coroutine?" — desde dos ángulos: la mitad de arriba es sobre **excepciones de negocio** (una llamada falla) y cómo `Job` vs `SupervisorJob` deciden si eso contamina a las coroutines hermanas. La mitad de abajo es sobre **cancelación** (alguien pidió parar) y cómo un `catch` mal escrito puede interferir con eso — es el mismo mecanismo de propagación de excepciones de Kotlin, pero con una excepción especial (`CancellationException`) que necesita tratamiento distinto.

## 2. Qué es y cómo funciona

Por default, un `Job` normal propaga las excepciones de forma agresiva: si una coroutine hija lanza una excepción no capturada, cancela a todas sus hermanas y se propaga hacia el `Job` padre (nodo B→D→C del diagrama). `SupervisorJob` es una variante donde el fallo de una hija no cancela a sus hermanas ni se propaga hacia arriba — cada hija falla de forma aislada, como si fueran tareas independientes bajo el mismo scope (nodo F→H, G no se entera).

`viewModelScope` usa internamente `SupervisorJob() + Dispatchers.Main.immediate`, justo por este motivo: un error cargando una sección de una pantalla no debería tirar abajo la carga de otra sección que corre en paralelo.

**Cancelación cooperativa** es un mecanismo distinto que usa el mismo canal (excepciones) para una cosa distinta (parar, no fallar). Cuando se llama `scope.cancel()`, Kotlin no "mata" la coroutine a la fuerza — le lanza una `CancellationException` en el próximo punto de suspensión, y espera que esa excepción se propague hacia afuera sin interferencia para que la coroutine termine limpio. El problema (nodo J del diagrama): `CancellationException` es una subclase de `Exception`, así que un `catch (e: Exception)` genérico la atrapa **también** a ella, no solo a los errores de negocio que el catch quería manejar. Si ese catch no relanza la `CancellationException` explícitamente, la cancelación queda "tragada" — la coroutine cree que manejó un error más y sigue ejecutando código que en realidad ya debería haber parado.

**`NonCancellable`** resuelve un caso puntual dentro de esta misma mecánica: código de limpieza que necesita correr *durante* una cancelación en curso — cerrar un archivo, revertir una escritura parcial, notificar que una operación se abortó. Una vez que una coroutine fue cancelada, cualquier punto de suspensión nuevo dentro de ella (otra llamada `suspend`) lanza `CancellationException` de inmediato — así que un `finally { someSuspendFun() }` normal no llega a completar esa limpieza si `someSuspendFun()` es suspendible. `withContext(NonCancellable) { }` abre una ventana que ignora el estado cancelado únicamente para lo que está adentro, dejando que ese código de limpieza puntual sí termine, sin revertir la cancelación del resto de la coroutine.

Cómo se relacionan ambos mecanismos: `SupervisorJob` decide **si** el fallo de una coroutine contamina a otras. La cancelación cooperativa decide **si** una coroutine individual respeta cuando le piden parar. Son ejes distintos, pero ambos dependen de que las excepciones se propaguen correctamente por la jerarquía de coroutines — y por eso un mismo error de código (un `catch` demasiado amplio) puede romper cualquiera de los dos.

## 3. Cómo se ve en distintos contextos

**App de fitness:** la pantalla de dashboard carga en paralelo el resumen semanal y el ranking de amigos, cada uno en su propio `launch` dentro de `viewModelScope`. Si el ranking falla porque el usuario no tiene amigos agregados todavía, el resumen semanal (que no depende de eso) se muestra igual — gracias a `SupervisorJob`, ese fallo no tumba la pantalla completa, solo esa sección queda en estado de error.

**App de e-commerce:** al salir de la pantalla de checkout a mitad de una validación de stock, el scope de esa pantalla se cancela. Si el código que valida el stock tiene un `catch (e: Exception)` genérico alrededor de la llamada de red sin relanzar `CancellationException`, esa validación sigue corriendo en segundo plano después de que el usuario ya volvió a la pantalla anterior — un caso típico de cancelación rota que no se nota hasta que aparece un crash o un estado inconsistente más adelante.

## 4. Implementación real

El PO pide: en la pantalla de Historial de Pedidos, cargar en paralelo el historial y el resumen de puntos de fidelidad — dos secciones independientes que pueden fallar por separado sin afectarse entre sí.

```kotlin
class OrdersViewModel(
    private val getOrderHistoryUseCase: GetOrderHistoryUseCase,
    private val getLoyaltyPointsUseCase: GetLoyaltyPointsUseCase
) {
    // SupervisorJob: el fallo de una sección no debe cancelar la otra
    private val viewModelScope = CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)

    private val _state = MutableStateFlow(OrdersState())
    val state: StateFlow<OrdersState> = _state.asStateFlow()

    fun onEvent(event: OrdersEvent) {
        when (event) {
            OrdersEvent.OnScreenEntered -> {
                loadOrderHistory()
                loadLoyaltyPoints()
            }
        }
    }

    private fun loadOrderHistory() {
        viewModelScope.launch {
            try {
                val orders = getOrderHistoryUseCase()
                _state.update { it.copy(orders = orders) }
            } catch (e: CancellationException) {
                throw e // la cancelación se relanza siempre, nunca se trata como un error de negocio
            } catch (e: Exception) {
                _state.update { it.copy(ordersError = e.message) }
            }
        }
    }

    private fun loadLoyaltyPoints() {
        viewModelScope.launch {
            try {
                val points = getLoyaltyPointsUseCase()
                _state.update { it.copy(loyaltyPoints = points) }
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                // este catch atrapa el error solo porque el scope está protegido con SupervisorJob;
                // si loadOrderHistory() falla sin manejo, esta coroutine sigue viva de todos modos
                _state.update { it.copy(loyaltyPointsError = e.message) }
            }
        }
    }

    fun onCleared() {
        viewModelScope.cancel() // dispara CancellationException en ambos launch activos
    }
}
```

El patrón `catch (e: CancellationException) { throw e }` antes del `catch (e: Exception)` genérico es lo que garantiza que, si `onCleared()` cancela el scope mientras `loadOrderHistory()` está a mitad de camino, esa cancelación se propague limpio en vez de quedar atrapada por el catch genérico y tratada como si fuera un error de red.

Si además hiciera falta registrar en analytics que la carga se cortó a mitad de camino (una llamada `suspend` propia), ese `finally` necesita la ventana de `NonCancellable`:

```kotlin
private fun loadOrderHistory() {
    viewModelScope.launch {
        try {
            val orders = getOrderHistoryUseCase()
            _state.update { it.copy(orders = orders) }
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            _state.update { it.copy(ordersError = e.message) }
        } finally {
            withContext(NonCancellable) {
                analyticsRepository.logLoadAttemptFinished() // suspend fun: sin NonCancellable, no llega a ejecutarse si la coroutine ya fue cancelada
            }
        }
    }
}
```

Sin `withContext(NonCancellable) { }`, si la coroutine llega al `finally` ya cancelada, `analyticsRepository.logLoadAttemptFinished()` lanzaría `CancellationException` en el instante en que se suspende — la llamada de analytics ni se completaría, sin ningún error visible que lo delate.

## 5. Buenas prácticas y errores comunes — checklist de auditoría

Si esta parte del código la entregó una IA, revisar puntualmente:

- **¿Hay un `catch (e: Exception)` (o peor, `catch (e: Throwable)`) alrededor de código suspendible sin relanzar `CancellationException`?** Es el error de cancelación cooperativa más común y más difícil de detectar en review — el código compila, el happy path funciona, y el bug solo aparece cuando alguien sale de la pantalla en el momento exacto en que esa coroutine está corriendo. La forma correcta es capturar `CancellationException` primero y relanzarla, o usar `catch (e: Exception) { if (e is CancellationException) throw e; ... }` si el código no puede tener dos bloques `catch` separados.
- **¿Usa `Job()` normal en vez de `SupervisorJob()` para operaciones independientes en paralelo?** Si el código arma un `CoroutineScope` manual (fuera de `viewModelScope`) con `Job()` a secas para lanzar varias operaciones que no dependen entre sí, un fallo en cualquiera de ellas va a cancelar a todas las demás — hay que verificar que el tipo de `Job` coincida con la intención real (aislar fallos vs propagarlos).
- **¿El `try/catch` envuelve un `async { }.await()` asumiendo que alcanza con eso para atrapar el error?** Con un `Job()` normal (no `SupervisorJob`), si otra coroutine hermana falla primero, el scope entero se cancela **antes** de que el flujo llegue al `catch` del `await()` — el catch nunca se ejecuta, aunque esté bien escrito. Esto solo funciona de forma predecible bajo `SupervisorJob`.
- **¿Usa `CoroutineExceptionHandler` como único mecanismo de manejo de errores?** Sirve como red de seguridad para excepciones realmente no anticipadas, pero no da forma de reflejar un error puntual en el campo correspondiente del `State` — no reemplaza el `try/catch` específico por operación. Además, `CoroutineExceptionHandler` no aplica a `async`: la excepción de un `async` queda guardada en el `Deferred` hasta que se llama `.await()`, ahí recién se relanza.
- **¿Cada `launch` paralelo tiene su propio manejo de error, o hay un único `try/catch` envolviendo varios `launch` a la vez?** Un solo `try/catch` no puede envolver múltiples `launch` independientes de forma útil — cada `launch` corre su propio bloque, así que el manejo de error tiene que vivir adentro de cada uno si se espera que fallen de forma aislada.
- **¿Hay una llamada `suspend` de limpieza dentro de un `finally` sin `withContext(NonCancellable) { }`?** Si la coroutine ya está cancelada al llegar al `finally`, cualquier punto de suspensión ahí adentro (sin esa ventana) lanza `CancellationException` de nuevo antes de completarse — la limpieza queda a medias, sin ningún error visible. Solo hace falta `NonCancellable` cuando ese código de limpieza es en sí mismo suspendible — código síncrono en el `finally` no lo necesita.