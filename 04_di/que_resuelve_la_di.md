# Qué resuelve la Inyección de Dependencias

## 1. Qué es

La Inyección de Dependencias (DI) es un patrón donde una clase **recibe** las dependencias que necesita en vez de **crearlas ella misma**. La responsabilidad de construir e "inyectar" esas dependencias se delega a algo externo (un framework de DI, o incluso construcción manual pasada por constructor).

No es una librería — es el patrón. Koin, Dagger/Hilt y Kodein son tres formas distintas de *implementar* ese patrón, pero el problema que resuelven es el mismo, y existe incluso sin usar ninguna de las tres (pasar dependencias por constructor a mano ya es DI).

## 2. El problema que resuelve

Sin DI, cada clase construye sus propias dependencias internamente:

```kotlin
class PlayerRepositoryImpl {
    private val dao = PlayerDao(DatabaseProvider.getInstance()) // la clase se arma sus propias dependencias
}
```

Esto genera tres problemas concretos:

- **Acoplamiento rígido**: `PlayerRepositoryImpl` ahora conoce `DatabaseProvider` y cómo se construye un `PlayerDao`. Si mañana cambia la forma de crear el DAO, hay que tocar cada clase que lo instancia a mano.
- **Imposible testear en aislamiento**: no hay forma de reemplazar `dao` por un fake en un test — está hardcodeado adentro. Para testear `PlayerRepositoryImpl` termina siendo necesario levantar una base de datos real.
- **Nadie sabe "quién construye qué"**: en un proyecto grande, si cada clase arma sus dependencias donde las necesita, no existe un lugar único donde ver el grafo de objetos de la app.

DI invierte esto: la clase declara qué necesita (por constructor), y algo externo se encarga de proveerlo.

```kotlin
class PlayerRepositoryImpl(private val dao: PlayerDao) : PlayerRepository {
    // la clase ya no sabe CÓMO se construye un PlayerDao, solo que lo recibe
}
```

## 3. Ejemplo mínimo comentado

```kotlin
// Sin DI: PlayerRepositoryImpl decide y construye su propia dependencia
class PlayerRepositoryImplSinDI : PlayerRepository {
    private val dao = PlayerDao(DatabaseProvider.getInstance()) // acoplado, no testeable
    override fun getPlayers(): Flow<List<Player>> = dao.getAll().map { it.toDomain() }
}

// Con DI: la dependencia entra por constructor, algo externo decide qué instancia pasar
class PlayerRepositoryImplConDI(
    private val dao: PlayerDao // "inyectada", no creada acá
) : PlayerRepository {
    override fun getPlayers(): Flow<List<Player>> = dao.getAll().map { it.toDomain() }
}

// En un test, ahora se puede pasar un fake sin tocar la clase real
class FakePlayerDao : PlayerDao {
    override fun getAll(): Flow<List<PlayerEntity>> = flowOf(emptyList())
}

val repositoryEnTest = PlayerRepositoryImplConDI(dao = FakePlayerDao())
```

La diferencia no es sintáctica — es que `PlayerRepositoryImplConDI` ya no decide **de dónde** viene su `PlayerDao`. Eso lo decide quien lo construye (un framework de DI, o directamente el test).

## 4. Matriz de criterio

| Escenario | Con DI (framework o manual) | Sin DI |
|---|---|---|
| Testear una clase en aislamiento | Se pasa un fake/mock por constructor sin tocar la clase | Requiere levantar la dependencia real (DB, red) o hacer la clase testeable a posteriori |
| Cambiar la implementación de una dependencia (ej: SQLDelight → Room) | Se cambia en un solo lugar (el módulo de DI) | Hay que tocar cada clase que instancia esa dependencia directamente |
| Proyecto chico, un solo desarrollador, prototipo descartable | Puede no justificarse el overhead de configurar un framework — DI manual por constructor alcanza | — |
| Proyecto que va a crecer o va a portfolio | Vale la pena un framework (Koin) desde el principio, más barato que migrar después | El costo de refactorizar todo a DI cuando ya creció es mucho mayor |

**El trade-off real no es "usar DI sí o no"** — DI manual (pasar dependencias por constructor sin ningún framework) ya es DI y en proyectos chicos puede ser suficiente. El framework (Koin, Dagger/Hilt) entra cuando el grafo de dependencias crece lo suficiente como para que armarlo a mano en cada punto de entrada de la app se vuelva repetitivo y propenso a error.

## 5. Caso trampa

**"Mi clase recibe la dependencia por constructor, así que ya está usando DI correctamente."**

Recibir por constructor es condición necesaria, pero no suficiente. Si en algún punto de la cadena alguien sigue haciendo esto:

```kotlin
class PlayersViewModel(
    private val repository: PlayerRepository = PlayerRepositoryImpl(PlayerDao(DatabaseProvider.getInstance()))
    //                                          ^ default value que construye la dependencia real acá adentro
) : ViewModel()
```

El constructor "acepta" la dependencia, pero el *default value* la sigue creando concretamente dentro de la clase. En un test, si no se pasa el parámetro explícitamente, se sigue instanciando `PlayerRepositoryImpl` real — el acoplamiento no desapareció, solo se escondió un nivel más adentro. DI real implica que **nada** dentro de la clase decide la implementación concreta, ni siquiera como valor por defecto.

## 6. Conexión con arquitectura real

En Timbax, `PlayerRepositoryImpl` implementa la interfaz `PlayerRepository` de `domain`, pero **quién decide** que en producción se use `PlayerRepositoryImpl` (contra SQLDelight) y no un fake es responsabilidad exclusiva del módulo de DI (Koin) — la clase que consume `PlayerRepository` (un `UseCase`, un `ViewModel`) nunca importa `PlayerRepositoryImpl` directamente, solo conoce la interfaz. Esto es lo que después hace posible, en `koin_fundamentos_y_scopes.md`, declarar `single<PlayerRepository> { PlayerRepositoryImpl(get()) }` en un solo lugar del grafo, sin que ninguna otra clase del proyecto sepa que esa implementación existe.