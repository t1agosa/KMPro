# kodein_alternativa.md

## 1. Qué es

Kodein (Kotlin Dependency Injection) es, al igual que Koin, un framework de DI multiplataforma que resuelve el grafo **en runtime**, 100% Kotlin puro. Comparte el mismo objetivo y la misma filosofía general que Koin — sin generación de código, sin anotaciones pesadas — pero con una diferencia de diseño central: Kodein construye su API apoyándose fuertemente en el **sistema de tipos genéricos de Kotlin** para resolver dependencias, mientras que Koin usa un enfoque más simple basado en un service locator con DSL declarativo.

## 2. El problema que resuelve (y por qué hoy pesa menos que Koin)

Kodein nació resolviendo el mismo problema genérico de DI que Koin — grafo runtime, sin atarse a Android, portable a KMP. Históricamente ofreció ventajas puntuales frente a las primeras versiones de Koin: mejor soporte de tipos genéricos complejos y, en algún momento, mejor rendimiento en la resolución de dependencias.

Hoy esa diferencia se diluyó: Koin maduró significativamente (mejor manejo de genéricos, mejor performance, soporte KMP más pulido) y absorbió gran parte de la comunidad KMP. El "problema" que Kodein resuelve sigue siendo válido en teoría, pero en la práctica Koin lo resuelve igual de bien con una API más simple y, sobre todo, con un ecosistema y una cantidad de recursos/documentación muchísimo mayor — lo cual importa tanto como la técnica en sí a la hora de elegir una librería para un proyecto real o de portfolio.

```kotlin
// Koin: DSL declarativo, se lee casi como configuración
val appModule = module {
    single<PlayerRepository> { PlayerRepositoryImpl(get()) }
}

// Kodein: se apoya en bindings tipados explícitos
val appModule = DI.Module("app") {
    bind<PlayerRepository>() with singleton { PlayerRepositoryImpl(instance()) }
}
```

## 3. Ejemplo mínimo comentado

```kotlin
import org.kodein.di.DI
import org.kodein.di.bind
import org.kodein.di.singleton
import org.kodein.di.instance

// Definición del módulo con Kodein
val appModule = DI.Module("appModule") {
    bind<PlayerDao>() with singleton { PlayerDao(instance()) }
    bind<PlayerRepository>() with singleton { PlayerRepositoryImpl(instance()) } // instance() resuelve PlayerDao automáticamente
}

// Contenedor DI principal
val di = DI {
    import(appModule)
}

// Consumo: se resuelve explícitamente por tipo, "by instance()"
class PlayersViewModel(di: DI) : DIAware {
    override val diContext = di.direct
    private val repository: PlayerRepository by instance()
}
```

La diferencia de fondo con el ejemplo de Koin (`koin_fundamentos_y_scopes.md`) no es solo estética: Kodein expone explícitamente `bind<Tipo>() with singleton { }`, apoyándose en el sistema de tipos para resolver — algo que en teoría da más seguridad en genéricos complejos, pero agrega verbosidad para el caso común (que es la mayoría de las dependencias de una app típica).

## 4. Matriz de criterio

| Criterio | Koin | Kodein |
|---|---|---|
| Adopción / comunidad KMP | Mayoría absoluta hoy, es el estándar de facto | Nicho, adopción baja y en descenso |
| Documentación y recursos de aprendizaje | Abundante, muchos ejemplos y tutoriales actuales | Escasa, documentación menos mantenida |
| Sintaxis para el caso común | DSL directo, menos verboso | Más explícito con tipos genéricos, más verboso |
| Manejo de genéricos complejos | Sólido en versiones actuales | Fue su ventaja histórica, hoy la brecha es mínima |
| Cuándo elegirlo hoy para un proyecto nuevo | Opción por defecto | Prácticamente nunca — solo si un proyecto legado ya lo usa y migrar no se justifica |

**El trade-off real hoy no es técnico, es de ecosistema**: la brecha funcional entre ambos se achicó, pero Koin ganó la adopción de la comunidad KMP. Elegir Kodein para un proyecto nuevo significa quedarse con menos recursos para resolver problemas futuros, sin una ventaja técnica lo suficientemente grande como para justificarlo.

## 5. Caso trampa

**"Si Kodein tiene mejor manejo de tipos genéricos, para un proyecto con arquitectura compleja debería ser la opción técnicamente superior."**

Esto fue cierto en un momento, pero evaluar una librería solo por una característica técnica aislada, ignorando adopción y mantenimiento del ecosistema, es un error de criterio común. Una librería con menos comunidad activa implica: menos respuestas en Stack Overflow/GitHub Issues ante un bug puntual, mayor riesgo de que el proyecto quede sin mantenimiento activo a mediano plazo, y menos ejemplos actualizados para versiones nuevas de Kotlin/Compose Multiplatform. Para un repo de portfolio o un proyecto real como Timbax, la pregunta correcta no es solo "¿cuál tiene mejor API en el paper?", sino "¿cuál tiene el respaldo de comunidad que me permite resolver problemas rápido cuando algo falla?".

## 6. Conexión con arquitectura real

Timbax usa Koin, no por default sin evaluar alternativas, sino porque frente a Kodein la decisión de criterio pesa más que la técnica pura: mismo objetivo (DI runtime multiplataforma), pero Koin ofrece menor fricción de sintaxis y mayor respaldo de comunidad para un proyecto real que necesita mantenerse en el tiempo. Es el mismo tipo de decisión que se documentará en `14_criterio_y_decisiones/`: elegir tecnología no es solo comparar features, es sopesar ecosistema, mantenimiento y curva de adopción del equipo (aunque sea un equipo de una sola persona).