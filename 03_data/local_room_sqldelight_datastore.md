# Local (Room / SQLDelight / DataStore)

## 1. Qué es

Es el conjunto de tecnologías de persistencia local disponibles en KMP, todas viviendo en `data/local`: **DataStore** (preferencias simples key-value, reactivo vía `Flow`), **Room** (soporte KMP oficial desde 2.7, queries con `@Dao`, nació Android-only), y **SQLDelight** (la más veterana pensada para multiplataforma desde el día uno, genera código Kotlin type-safe a partir de archivos `.sq`). Las tres cumplen el mismo rol arquitectónico — ser la fuente de datos local que el `RepositoryImpl` orquesta — pero resuelven problemas de persistencia distintos y no son intercambiables entre sí.

```kotlin
// SQLDelight — archivo Player.sq define el schema Y las queries
CREATE TABLE PlayerEntity (
    id TEXT NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    score INTEGER NOT NULL
);

getAll:
SELECT * FROM PlayerEntity;

insertPlayer:
INSERT OR REPLACE INTO PlayerEntity(id, name, score) VALUES (?, ?, ?);
```

## 2. El problema que resuelve

Antes de que existieran opciones KMP-nativas para persistencia estructurada, un proyecto multiplataforma tenía que implementar la capa de guardado local por separado en cada plataforma (Room en Android, Core Data o SQLite crudo en iOS), duplicando el mapeo Entity ↔ tabla y arriesgando que la lógica de queries divergiera entre plataformas con el tiempo. SQLDelight y Room-KMP resuelven esto generando, a partir de una única fuente (`.sq` o entidades `@Entity` anotadas en `commonMain`), el código de acceso a datos que corre igual en las tres plataformas — el `Dao`/queries se escriben una sola vez. DataStore resuelve un problema más chico pero igual de recurrente: guardar 2-3 valores sueltos (¿está logueado el usuario? ¿qué tema eligió?) sin necesitar el peso de una tabla SQL completa para eso.

## 3. Ejemplo mínimo comentado

**SQLDelight (usado en Timbax):**

```kotlin
// Player.sq — define schema + queries, el compilador genera PlayerQueries
CREATE TABLE PlayerEntity (
    id TEXT NOT NULL PRIMARY KEY,
    name TEXT NOT NULL,
    score INTEGER NOT NULL
);

getAll:
SELECT * FROM PlayerEntity;

insertPlayer:
INSERT OR REPLACE INTO PlayerEntity(id, name, score) VALUES (?, ?, ?);

// data/local — el DAO-like wrapper sobre las queries generadas
class PlayerDao(private val queries: PlayerQueries) {

    fun getAll(): Flow<List<PlayerEntity>> =
        queries.getAll().asFlow().mapToList(Dispatchers.IO)

    suspend fun insertAll(players: List<PlayerEntity>) {
        players.forEach { queries.insertPlayer(it.id, it.name, it.score) }
    }
}
```

**DataStore (para settings sueltos, ej. tema elegido en Timbax):**

```kotlin
class SettingsDataStore(private val dataStore: DataStore<Preferences>) {

    private val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")

    val isDarkMode: Flow<Boolean> =
        dataStore.data.map { prefs -> prefs[DARK_MODE_KEY] ?: false }

    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs -> prefs[DARK_MODE_KEY] = enabled }
    }
}
```

## 4. Matriz de criterio

**SQLDelight:**
- Usar cuando: el proyecto es KMP-first desde el diseño (como Timbax) y necesitás datos estructurados con relaciones, queries complejas, o cuando la madurez multiplataforma real (sin fricción histórica Android-only) pesa más que la familiaridad con anotaciones estilo Room.
- NO usar cuando: el equipo viene 100% del mundo Android y prefiere la sintaxis de anotaciones (`@Entity`, `@Dao`) por sobre archivos `.sq` separados — la curva de aprendizaje de SQL explícito en `.sq` puede ser fricción inicial para alguien acostumbrado a Room.
- Trade-off: SQL explícito en archivos `.sq` da control total y queries type-safe generadas, pero es un lenguaje/sintaxis distinto al resto del código Kotlin — se pierde el "todo es Kotlin" que sí tiene Room.

**Room (KMP desde 2.7):**
- Usar cuando: el proyecto es Android-first con KMP como plus, y el equipo ya tiene experiencia profunda con Room de proyectos Android previos — reduce fricción por familiaridad con el ecosistema Google.
- NO usar cuando: el proyecto es KMP-first puro — Room nació Android-only y, aunque hoy soporta KMP, todavía tiene menos historial de madurez multiplataforma que SQLDelight.
- Trade-off: anotaciones Kotlin (`@Entity`, `@Dao`) en vez de un lenguaje aparte, más cercano a "todo es Kotlin", pero con menos años de rodaje fuera de Android que la alternativa veterana.

**DataStore:**
- Usar cuando: el dato es simple, key-value, sin necesidad de queries ni relaciones — flags de configuración, preferencias de usuario, tokens de sesión.
- NO usar cuando: el dato tiene estructura (múltiples campos relacionados, listas, necesidad de queries) — ahí DataStore se queda corto y conviene SQLDelight/Room desde el principio, no como "para ya, después migro".
- Trade-off: API mínima y reactiva vía `Flow`, evolución directa de SharedPreferences — pero no está pensado para nada que necesite ser consultado con filtros o relacionado entre sí.

## 5. Caso trampa

Usar DataStore para guardar una lista que crece con el tiempo, en vez de usarlo para lo que fue diseñado:

```kotlin
// ❌ trampa: guardar una lista completa de jugadores serializada como JSON en DataStore
class PlayerDataStore(private val dataStore: DataStore<Preferences>) {

    private val PLAYERS_KEY = stringPreferencesKey("players_json")

    val players: Flow<List<Player>> = dataStore.data.map { prefs ->
        val json = prefs[PLAYERS_KEY] ?: "[]"
        Json.decodeFromString<List<Player>>(json) // deserializa TODA la lista en cada lectura
    }

    suspend fun addPlayer(player: Player) {
        dataStore.edit { prefs ->
            val current = Json.decodeFromString<List<Player>>(prefs[PLAYERS_KEY] ?: "[]")
            prefs[PLAYERS_KEY] = Json.encodeToString(current + player) // reescribe TODO el blob
        }
    }
}
```

Funciona con 5 jugadores. El problema aparece con escala: cada `addPlayer()` reescribe el archivo completo de preferencias (no hay operación de "insertar una fila", porque DataStore no tiene el concepto de fila), y cada lectura deserializa el JSON entero en memoria aunque la UI solo necesite mostrar 3 registros. Además, se pierde toda capacidad de query (¿cómo pedís "el jugador con más score" sin traer la lista completa y filtrarla en Kotlin?) — algo que una tabla SQL resuelve nativamente con una query indexada.

La señal de alarma: si en algún punto aparece un `List<T>` serializado completo como un solo valor de DataStore, es una tabla disfrazada de preferencia — la solución es migrar ese dato a SQLDelight/Room desde el principio, no esperar a que el JSON crezca lo suficiente como para notar el problema de performance.

## 6. Conexión

En Timbax, `PlayerDao` (documentado también en `repository_impl.md`) está construido sobre SQLDelight — es la fuente local que `PlayerRepositoryImpl` observa como single source of truth vía `Flow`, y a la que los mappers de `dto_entity_mappers.md` traducen desde `PlayerEntity` hacia `Player`. DataStore, en cambio, cubre un rol distinto y más chico dentro de la misma app: configuración de usuario (tema, preferencias de sonido) que no necesita ni merece el peso de una tabla relacional. Ambas conviven en el mismo proyecto porque resuelven problemas de persistencia de naturaleza distinta, no porque una sea "mejor" que la otra en abstracto.