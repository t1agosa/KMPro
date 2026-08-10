# dagger_hilt_vs_koin.md

## 1. Qué es

Dagger es un framework de DI para JVM que resuelve el grafo de dependencias **en compile-time**, generando código Java/Kotlin real mediante annotation processing (hoy vía KSP). Hilt es una capa sobre Dagger, hecha por Google específicamente para Android, que simplifica su configuración atándose a los componentes del ciclo de vida de Android (`Activity`, `Fragment`, `ViewModel`).

La diferencia de fondo con Koin no es de sintaxis — es **cuándo** se resuelve el grafo: Dagger/Hilt lo resuelve el compilador, generando clases concretas antes de que la app corra; Koin lo resuelve en runtime, con un mapa de fábricas (`lambdas`) que se consulta cuando algo pide una dependencia.

## 2. El problema que resuelve (y por qué encaja mal en KMP)

Dagger/Hilt resuelve el mismo problema genérico de DI (`que_resuelve_la_di.md`), pero con una garantía extra: si falta una dependencia o hay un ciclo en el grafo, **no compila** — el error aparece antes de ejecutar la app, no en runtime como en Koin.

El problema para KMP es estructural, no de calidad de la librería: Hilt está construido sobre las anotaciones y componentes del ciclo de vida de **Android** (`@AndroidEntryPoint`, `@HiltViewModel`, `ViewComponent`, `ActivityComponent`). Esas piezas no existen en iOS, Desktop ni Web — no hay `Activity` ni `Fragment` fuera de Android. Dagger "pelado" (sin Hilt) sí es JVM puro y en teoría podría compilar para Desktop, pero pierde toda la ergonomía que lo hace útil, y sigue sin correr en iOS porque el annotation processing con KSP no está pensado para Kotlin/Native de la misma forma.

```kotlin
// Esto es válido en un proyecto Android-only con Hilt...
@HiltViewModel
class PlayersViewModel @Inject constructor(
    private val getPlayersUseCase: GetPlayersUseCase
) : ViewModel()

// ...pero @HiltViewModel y @Inject constructor no tienen equivalente
// funcional en iosMain o desktopMain: el ciclo de vida que Hilt asume
// (Activity/Fragment/ViewModelStoreOwner de Android) no existe ahí.
```

## 3. Ejemplo mínimo comentado (comparación lado a lado)

```kotlin
// ---- KOIN (multiplataforma, runtime) ----
// Se declara en un módulo común, corre igual en Android/iOS/Desktop
val appModule = module {
    single<PlayerRepository> { PlayerRepositoryImpl(get()) }
    viewModel { PlayersViewModel(get()) }
}

// ---- DAGGER/HILT (Android-only, compile-time) ----
// La dependencia se declara en un @Module, atado a un @InstallIn de Android
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun providePlayerRepository(dao: PlayerDao): PlayerRepository =
        PlayerRepositoryImpl(dao)
}

// El ViewModel se marca con @HiltViewModel, algo que solo tiene sentido
// dentro del ciclo de vida de Android
@HiltViewModel
class PlayersViewModel @Inject constructor(
    private val repository: PlayerRepository
) : ViewModel()
```

La diferencia visible es que Koin es una función Kotlin normal (`module { }`) que corre en cualquier target; Hilt depende de anotaciones que el compilador de Android procesa contra clases del framework de Android — no hay forma de que ese mismo código compile en `iosMain`.

## 4. Matriz de criterio

| Criterio | Koin | Dagger/Hilt |
|---|---|---|
| Multiplataforma (Android + iOS + Desktop) | Sí, nativo | No — Hilt es Android-only; Dagger puro no cubre iOS |
| Detección de errores | Runtime (crashea al ejecutar si falta algo) | Compile-time (no compila si el grafo está incompleto) |
| Velocidad de build | Sin generación de código, build más simple | Genera código en cada compilación (KSP), puede alentar el build en proyectos grandes |
| Curva de aprendizaje / setup | Baja, sintaxis Kotlin directa | Más alta, requiere entender componentes, scopes y anotaciones propias de Android |
| Cuándo SÍ tendría sentido evaluarlo | — | Proyecto **Android-only** que no tiene planes reales de expandirse a KMP, y donde la seguridad de compile-time pesa más que la portabilidad |
| Proyecto KMP real (como Timbax) | Elección correcta por defecto | Fuera de discusión — rompería la posibilidad de compartir el módulo `presentation` con iOS |

**El trade-off no es "Koin es mejor que Dagger" en abstracto** — en un proyecto puramente Android, la seguridad de compile-time de Dagger/Hilt es una ventaja real que Koin no tiene. La decisión se invierte específicamente porque el proyecto es KMP: ahí Hilt deja de ser una opción viable, no solo una preferencia.

## 5. Caso trampa

**"Podemos usar Hilt solo en el módulo Android y Koin en el resto, así aprovechamos lo mejor de cada uno."**

Suena razonable, pero rompe la premisa central de KMP: el `ViewModel` (capa `presentation`) vive en `commonMain` precisamente para compartirse entre plataformas. Si se inyecta con `@HiltViewModel`, ese `ViewModel` queda atado a Android y ya no puede compilarse ni usarse desde `iosMain` — dejaría de ser código compartido. Mezclar Hilt "solo para Android" dentro de un módulo pensado como multiplataforma reintroduce exactamente el acoplamiento que Clean Architecture + KMP buscan evitar: una capa que debería ser agnóstica de plataforma termina dependiendo de anotaciones de Android.

Donde sí es válido usar Dagger/Hilt en un proyecto KMP: exclusivamente dentro de `androidMain`, para dependencias que ya son 100% específicas de Android (por ejemplo, un `WorkManager` o un servicio del sistema) — nunca para algo que vive en `commonMain`.

## 6. Conexión con arquitectura real

Timbax usa Koin en `commonMain` precisamente porque el `PlayersViewModel` necesita compilar y funcionar igual en Android e iOS — es el mismo argumento que ya se vio en `koin_fundamentos_y_scopes.md`, pero acá queda claro **por qué Hilt nunca fue una opción real** para ese módulo: no es una preferencia de gusto, es una restricción técnica derivada de dónde vive el código (`commonMain` vs `androidMain`).