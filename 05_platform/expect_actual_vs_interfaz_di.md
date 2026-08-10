# expect_actual_vs_interfaz_di.md

## 1. Qué es

Es la decisión de criterio entre dos mecanismos que resuelven el mismo problema de fondo — "necesito código distinto por plataforma" — pero con garantías y costos distintos: `expect/actual` (resolución en compile-time, sin indirección) vs. una interfaz común en `domain`/`data` con una implementación concreta por plataforma, inyectada vía Koin (resolución en runtime, a través del grafo de DI).

```kotlin
// Opción A: expect/actual
expect fun getDeviceId(): String

// Opción B: interfaz + DI
interface DeviceIdProvider {
    fun getDeviceId(): String
}
// androidMain: single<DeviceIdProvider> { AndroidDeviceIdProvider(androidContext()) }
// iosMain: single<DeviceIdProvider> { IosDeviceIdProvider() }
```

## 2. El problema que resuelve

Sin este criterio explícito, es fácil caer en dos errores simétricos: usar `expect/actual` para todo lo que difiere por plataforma (incluyendo cosas con dependencias o que necesitan fakes en tests, donde se vuelve incómodo) o usar interfaz+DI para todo (incluyendo casos triviales de una sola función pura, donde agregar una interfaz y un binding de Koin es ceremonia de más para nada). Ninguno de los dos mecanismos es "el correcto" en abstracto — la pregunta que resuelve este archivo es *cuál conviene según la forma del problema concreto*, y esa pregunta no está resuelta en ningún lado de la fuente original ni es intuitiva la primera vez que se topa con el tema.

## 3. Ejemplo mínimo comentado

```kotlin
// Caso simple: candidato ideal para expect/actual — sin dependencias, sin estado
expect fun getPlatformName(): String
```

```kotlin
// Caso con dependencias: mal candidato para expect/actual, bien para interfaz + DI
interface AnalyticsLogger {
    fun logEvent(name: String, params: Map<String, String>)
}

// androidMain — necesita el Context de Android
class AndroidAnalyticsLogger(private val context: Context) : AnalyticsLogger {
    override fun logEvent(name: String, params: Map<String, String>) { /* ... */ }
}

// Koin, androidMain
single<AnalyticsLogger> { AndroidAnalyticsLogger(androidContext()) }
```

Con `expect/actual`, pasarle el `Context` a un `actual class AndroidAnalyticsLogger` implica declarar el `expect class` con constructor, y ya empieza a sentirse forzado comparado con dejar que Koin resuelva la dependencia naturalmente.

## 4. Matriz de criterio

**Usar `expect/actual` cuando:**
- Es una función (no una clase con estado) sin dependencias externas.
- El comportamiento es realmente fijo por plataforma, sin necesidad de swappear implementaciones en tests o en distintos flavors.
- Querés la garantía dura de compile-time: si te olvidás el `actual`, no compila — con DI, un `single` mal registrado recién explota en runtime al resolver el grafo.

**Usar interfaz + DI cuando:**
- La implementación necesita recibir dependencias (Context, un cliente ya configurado, otra clase del grafo).
- Necesitás testear la lógica que *consume* esa capacidad con un fake (`FakeAnalyticsLogger`) sin tocar plataforma real — esto es imposible de forma limpia con `expect/actual`, porque no hay contrato inyectable, solo una función global.
- Podrías necesitar más de una implementación en la misma plataforma (ej: un logger real y uno no-op según build variant) — Koin permite resolver eso con distintos módulos o calificadores; `expect/actual` no tiene ese grado de libertad, es una implementación fija por target.
- Ya tenés un ViewModel o UseCase que depende de la capacidad — mantener el patrón de inyección de dependencias consistente (todo entra por constructor) es más legible que mezclar "esto se inyecta" con "esto se llama como función global de `expect/actual`".

**Trade-off real:** `expect/actual` gana en simplicidad y en la garantía de compile-time; interfaz+DI gana en testabilidad y flexibilidad de composición. La regla mental rápida: *si necesitás pasarle algo al constructor de la implementación, ya no es candidato para `expect/actual`*.

## 5. Caso trampa

Tenés `expect fun getDeviceId(): String` funcionando bien. Un sprint después, Marketing pide que el ID de dispositivo se loguee junto con la versión del SO y se cachee para no recalcularlo en cada llamada. La reacción rápida es agregar un `expect class DeviceInfoProvider` con estado interno (`private var cachedId: String? = null`) y mantener `expect/actual`, porque "ya estaba así, solo lo extiendo". El problema: ahora esa clase tiene estado y potencialmente vas a querer testear el comportamiento de cacheo (¿se recalcula si cambia el SO? ¿se limpia en algún momento?) sin depender de la implementación real de Android/iOS — y `expect/actual` no te da un punto de inyección para un fake en ese test. La señal de que cruzaste la línea "función pura → estado con reglas" es exactamente la señal de que tendría que migrar a interfaz + DI, aunque signifique un refactor de lo que ya estaba andando.

## 6. Conexión con arquitectura real

En Timbax, el caso de uso típico de interfaz+DI en vez de `expect/actual` sería un `FirebaseAuthProvider` o cualquier wrapper sobre el SDK de GitLive Firebase que necesita configuración (API keys, instancia de la app de Firebase) — ahí ya hay dependencias de por medio, así que va como interfaz en `domain` (o `data`, según si otras capas necesitan conocerla) con implementación inyectada por Koin, siguiendo el mismo patrón que `PlayerRepository`/`PlayerRepositoryImpl`. `expect/actual` puro queda reservado a lo verdaderamente atómico, como `getPlatformName()` del archivo anterior.