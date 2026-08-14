# Tracking de eventos y analytics

## 1. Qué es

Analytics es registrar **acciones del usuario** dentro de la app (qué pantalla vio, qué botón tocó, si completó un flujo) para entender comportamiento agregado — a diferencia de logging (que registra eventos técnicos para debuggear) y telemetry (que mide duración de procesos). En KMP, vía GitLive Firebase SDK, se usa `Firebase.analytics.logEvent(name) { param(...) }`: una API DSL que manda el evento con parámetros clave-valor al backend de Firebase/Google Analytics 4.

## 2. El problema que resuelve

Sin analytics, las decisiones de producto se toman a ciego: "¿la gente usa la función de reenganche?", "¿en qué paso del flujo de creación de partida abandonan?" son preguntas que sin datos agregados solo se responden con intuición. Analytics permite construir funnels (secuencias de eventos esperados) y ver dónde se cae el usuario real, además de segmentar comportamiento por versión de app, plataforma, o cualquier parámetro custom que definas.

## 3. Ejemplo mínimo comentado

```kotlin
// commonMain — evento custom con parámetros
import dev.gitlive.firebase.Firebase
import dev.gitlive.firebase.analytics.analytics
import dev.gitlive.firebase.analytics.logEvent

fun trackGameSaved(gameType: String, playerCount: Int) {
    Firebase.analytics.logEvent("game_saved") {
        param("game_type", gameType)     // string
        param("player_count", playerCount.toLong()) // numérico
    }
}

// evento predefinido de Firebase (aprovecha reportes ya armados en la consola)
fun trackScreenView(screenName: String) {
    Firebase.analytics.logEvent("screen_view") {
        param("screen_name", screenName)
    }
}
```

```kotlin
// user property: describe al usuario, no una acción puntual — vive fuera de un evento
Firebase.analytics.setUserProperty("preferred_game", "chinchon")
```

## 4. Matriz de criterio

| Escenario | Usar | NO usar | Trade-off |
|---|---|---|---|
| Acción concreta del usuario (tocó, completó, abandonó) | `logEvent` custom con nombre descriptivo (`game_saved`) | Nombres genéricos (`click`, `action`) repetidos por todas partes | Nombres genéricos hacen imposible diferenciar qué se tocó realmente en los reportes |
| Evento cubierto por el catálogo estándar de Firebase (`screen_view`, `select_item`) | Evento predefinido | Reinventar tu propio nombre custom | Los predefinidos activan reportes y funnels ya armados en la consola sin config extra |
| Dato que describe al usuario, no una acción puntual (segmento, tier) | `setUserProperty` | Mandarlo como parámetro de cada evento | User property persiste y segmenta reportes sin repetirlo en cada log |
| Cualquier dato personal identificable (email, nombre real) | Nunca, ni como parámetro ni como user property | — | Viola compliance de privacidad — Firebase Analytics no está pensado para PII |
| Definir el catálogo de eventos del proyecto | Un `object AnalyticsEvents` centralizado con constantes | Strings sueltos hardcodeados en cada call site | Evita typos que crean eventos duplicados por error de tipeo (`game_saved` vs `gamesaved`) |

## 5. Caso trampa

**"Agregué `param()` con mis datos custom, van a aparecer como columnas filtrables en el reporte del evento."**

No siempre es automático — hay un issue real documentado en el repositorio de GitLive (`firebase-kotlin-sdk`, issue #691) en versiones del SDK donde los parámetros custom pasados dentro del builder `logEvent { param(...) }` terminaban anidados bajo nombres internos raros (`v5_1`, `internalMap_1`) en vez de aparecer como parámetros individuales del evento en Firebase DebugView — rompiendo el filtrado por esos parámetros en la consola. La lección de fondo: **siempre verificar en Firebase DebugView** que un evento nuevo llega con la forma esperada antes de asumir que "ya está trackeado", en vez de confiar ciegamente en que el código compiló y no tiró excepción.

Trampa relacionada, de límites del propio Firebase (no de GitLive): los nombres de evento tienen tope de 500 distintos por proyecto y 40 caracteres, no pueden empezar con `firebase_`/`google_`/`ga_`, y cada evento admite máximo 25 parámetros — si no centralizás los nombres en un solo lugar del código, es fácil terminar generando variantes por typo (`gameSaved` vs `game_saved`) que cuentan como dos eventos distintos y consumen cupo del límite de 500 sin que nadie lo note hasta que se acerca al techo.

## 6. Conexión con Timbax

Timbax ya usa Firebase vía GitLive, así que `Firebase.analytics` está a un import de distancia sin infraestructura nueva. Un catálogo mínimo con sentido para arrancar: `game_saved` (con `game_type` y `player_count` como parámetros, conectando directo con el `SaveScoreUseCase` que ya venimos usando de ejemplo en todo el repo), `screen_view` en cada pantalla principal, y quizás un `game_abandoned` si el usuario sale de una partida en curso sin guardar — ese último es justo el tipo de dato que solo un funnel de analytics puede revelar, no la intuición de producto.