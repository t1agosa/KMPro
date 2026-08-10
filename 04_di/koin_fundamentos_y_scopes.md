# koin_fundamentos_y_scopes.md

## 1. Qué es

Koin es el framework de Inyección de Dependencias más usado en KMP. A diferencia de Dagger/Hilt, resuelve el grafo de dependencias **en runtime** (no genera código en compile-time vía KSP/annotation processing) — es 100% Kotlin puro, sin reflexión pesada ni generación de código, lo que lo hace liviano y portable a cualquier target de KMP (Android, iOS, Desktop, Web).

Se organiza en **módulos** (`module { }`), donde cada dependencia se declara con un **scope** que define su ciclo de vida: `single`, `factory` o `viewModel`.

## 2. El problema que resuelve

Ya vimos en `que_resuelve_la_di.md` el problema genérico de DI. Koin resuelve, puntualmente, **cómo construir el grafo completo de la app sin acoplarse a Android** (a diferencia de Hilt) y **sin pagar el costo de compile-time** (a diferencia de Dagger). Sin un framework, cada punto de entrada de la app (el `MainActivity`, el `App.swift`, el entrypoint de Desktop) tendría que ensamblar a mano toda la cadena de dependencias:

```kotlin
// Sin Koin: ensamblar el grafo a mano en cada entrypoint
val dao = PlayerDao(DatabaseProvider.getInstance())
val repository = PlayerRepositoryImpl(dao)
val getPlayersUseCase = GetPlayersUseCase(repository)
val viewModel = PlayersViewModel(getPlayersUseCase)
```

Esto se vuelve insostenible cuando hay decenas de dependencias y varias pantallas — cada una necesitaría repetir parte de esta cadena. Koin centraliza esa construcción en módulos declarativos, y resuelve el grafo automáticamente cuando algo pide una dependencia.

## 3. Ejemplo mínimo comentado

```kotlin
// Definición del módulo (comúnmente en di/AppModule.kt)
val appModule = module {
    single<PlayerRepository> { PlayerRepositoryImpl(get()) } // singleton, una sola instancia en toda la app
    single { PlayerDao(get()) }                               // get() resuelve la dependencia de PlayerDao automáticamente
    factory { GetPlayersUseCase(get()) }                       // nueva instancia cada vez que se pide
    viewModel { PlayersViewModel(get()) }                      // scope atado al ciclo de vida del ViewModel
}

// Arranque de Koin (una sola vez, en el entrypoint de cada plataforma)
fun initKoin() {
    startKoin {
        modules(appModule)
    }
}

// Consumo dentro de un composable (Android/KMP)
@Composable
fun PlayersScreen(viewModel: PlayersViewModel = koinViewModel()) {
    // Koin resuelve automáticamente toda la cadena: 
    // PlayersViewModel <- GetPlayersUseCase <- PlayerRepository <- PlayerDao
}
```

`get()` dentro de un módulo le dice a Koin "resolvé esta dependencia usando lo que ya está declarado en el grafo" — no hace falta pasarla a mano, Koin la busca por tipo.

## 4. Matriz de criterio

| Scope | Cuándo usarlo | Cuándo NO usarlo |
|---|---|---|
| `single` | Dependencias que deben ser una única instancia compartida en toda la app: `HttpClient`, `Database`, `Repository` (si no tiene estado por-pantalla) | Si necesitás una instancia nueva cada vez (ej: un `UseCase` con estado interno mutable, poco común pero posible) |
| `factory` | Objetos livianos que no tiene sentido compartir — típicamente `UseCase`s sin estado propio | Si el objeto es costoso de crear (ej: una conexión de red) — ahí conviene `single` |
| `viewModel` | Específico para ViewModels — Koin lo ata al ciclo de vida correcto de la plataforma (`ViewModelStoreOwner` en Android, equivalente en KMP) | Para cualquier clase que no sea un ViewModel — usar `single`/`factory` según corresponda |

**Trade-off real de Koin vs Dagger/Hilt** (se profundiza en `dagger_hilt_vs_koin.md`): Koin resuelve el grafo en runtime, lo que significa que un error de dependencia faltante **no se detecta hasta ejecutar la app** (crash en runtime), mientras que Dagger lo detecta en compile-time. Es el precio que se paga por la portabilidad multiplataforma y la simplicidad de setup.

## 5. Caso trampa

**"Declaré la dependencia en el módulo, así que ya está inyectada correctamente."**

Declarar en el módulo no alcanza si el **scope elegido es el incorrecto**. Un error común: usar `single` para algo que en realidad necesita ser `factory`.

```kotlin
// Trampa: un UseCase declarado como single que internamente guarda estado mutable
single { ValidateScoreUseCase() } // si ValidateScoreUseCase tiene un `var lastError: String? = null` interno,
                                    // ese estado se comparte entre TODAS las pantallas que lo usen simultáneamente
```

Si `ValidateScoreUseCase` guarda estado mutable interno y se declara `single`, ese estado queda compartido entre todos los lugares de la app que lo inyecten — un bug de concurrencia difícil de rastrear, porque el código "compila y funciona" en el caso simple de una sola pantalla usándolo a la vez. La regla práctica: si la clase tiene estado mutable propio, casi nunca debería ser `single` salvo que ese estado compartido sea intencional (como un `Repository` con cache en memoria).

## 6. Conexión con arquitectura real

En Timbax, el módulo de Koin es el único lugar del proyecto donde se importa `PlayerRepositoryImpl` (la implementación concreta) junto con `PlayerRepository` (la interfaz de `domain`) — ninguna otra clase conoce ambas cosas a la vez. Esto es la continuación directa de lo que se explicó en `que_resuelve_la_di.md`: el módulo de DI es el punto de conexión entre la capa `data` y la capa `domain`, respetando la Dependency Rule sin que ninguna clase de negocio importe algo de infraestructura.