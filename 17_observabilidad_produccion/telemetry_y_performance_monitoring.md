# Telemetry y performance monitoring

## 1. Qué es

Performance monitoring es medir cuánto tarda tu app en hacer cosas concretas (arrancar, cargar una pantalla, responder a un tap, completar un request de red) y mandar esos números a un backend donde se agregan por versión de app, dispositivo, país, etc. — para detectar degradación de performance en producción antes de que se convierta en una mala review. Se instrumenta con **traces**: un trace tiene un punto de inicio, un punto de fin, y opcionalmente atributos (strings para filtrar) y métricas custom (números para graficar). A diferencia del logging (que registra eventos discretos), telemetry mide **duración** y **frecuencia** de procesos.

## 2. El problema que resuelve

Sin telemetry, "la app se siente lenta" es una queja sin datos: no sabés si es un dispositivo específico, una versión de Android vieja, un endpoint de red lento, o una query de base de datos que se degradó con más filas. Los out-of-the-box traces (arranque de app, requests HTTP) dan una primera foto automática, pero no cubren procesos específicos de tu dominio — eso requiere instrumentación manual, a propósito, en los puntos que vos identificás como críticos.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — trace custom sobre el flujo de guardado de partida
import dev.gitlive.firebase.Firebase
import dev.gitlive.firebase.perf.performance

suspend fun saveGameWithTrace(players: List<Player>) {
    val trace = Firebase.performance.newTrace("save_game_flow")
    trace.start()
    trace.putAttribute("player_count", players.size.toString())

    try {
        saveGameUseCase(players)
        trace.incrementMetric("success")
    } catch (e: Exception) {
        trace.incrementMetric("failure")
        throw e
    } finally {
        trace.stop() // siempre se cierra, haya fallado o no
    }
}
```

```kotlin
// medición de un request HTTP puntual (fuera del tracking automático de Ktor)
val metric = Firebase.performance.newHttpMetric("https://api.timbax.com/sync", "POST")
metric.start()
val response = httpClient.post("https://api.timbax.com/sync") { /* ... */ }
metric.setHttpResponseCode(response.status.value)
metric.stop()
```

## 4. Matriz de criterio

| Escenario | Usar | NO usar | Trade-off |
|---|---|---|---|
| Flujo de negocio crítico (guardar partida, sincronizar) | Trace custom con `start()`/`stop()` alrededor del flujo completo | Confiar solo en los traces automáticos | Los automáticos no conocen tu dominio, solo miden lo genérico (arranque, red) |
| Filtrar performance por condición (ej: modo offline, feature flag) | `putAttribute()` en el trace | Crear un trace distinto por cada variante | Attributes filtran en el dashboard sin explotar la cantidad de traces distintos a mantener |
| Contar ocurrencias dentro de un trace (cache hits, reintentos) | `incrementMetric()` | Loguear cada ocurrencia por separado con Kermit | Metric se agrega y grafica en el dashboard; un log individual no |
| Medición de latencia de un solo evento puntual y aislado | Trace simple sin mucho aparato | Instrumentar todo el codebase con traces | Sobre-instrumentar agrega overhead de mantenimiento sin ganancia — no todo necesita medirse |
| Detectar network requests lentos | `newHttpMetric` o el tracking automático del SDK | Medir tiempo de red "a mano" con timestamps propios | El SDK ya normaliza y agrega esos datos en el dashboard con contexto de dispositivo/versión |

## 5. Caso trampa

**"Instrumenté un trace en `saveGameWithTrace()` y no veo nada en el dashboard de Firebase, algo está roto."**

No necesariamente — Firebase Performance Monitoring tiene una demora real de agregación de datos (puede tardar minutos, a veces bastante más, en aparecer en la consola), especialmente en builds de debug donde el pipeline de recolección es menos prioritario. Antes de asumir que la instrumentación está mal, hay que dar tiempo y, si es posible, verificar con una build de release o con el modo de debug logging del SDK de Performance, no solo mirando el dashboard en caliente.

Trampa relacionada: un trace que arranca con `start()` pero cuyo `stop()` vive solo en el camino feliz (sin `finally` o equivalente) queda "colgado" si el proceso lanza una excepción antes de llegar al `stop()` — el trace nunca se cierra, no aparece con datos útiles, y en el peor caso podés terminar con falsos negativos (parece que el flujo nunca se ejecuta cuando en realidad está fallando silenciosamente antes de tiempo).

## 6. Conexión con Timbax

Timbax ya trae Firebase vía GitLive, y el SDK expone `firebase-perf` con la misma API mostrada arriba (`Firebase.performance.newTrace`), sin necesitar código `expect/actual` adicional. El candidato más directo para el primer trace real: envolver el flujo completo de `SaveScoreUseCase` (desde que el usuario confirma hasta que el repository confirma el guardado en SQLDelight/Firestore), con `player_count` como atributo — así, si en producción aparece que guardar tarda de más para partidas con muchos jugadores, el dato sale del dashboard en vez de una sospecha sin evidencia.