# resources_multiplatform.md

## 1. Qué es

`composeResources` es el sistema oficial de recursos compartidos de Compose Multiplatform: strings, imágenes (`.xml`/`.png`/`.svg`/`.webp`), fuentes (`.ttf`/`.otf`) y archivos raw, definidos una sola vez en `commonMain` bajo una carpeta `composeResources/`, con el compilador generando accesos type-safe (`Res.string.app_name`, `Res.drawable.icon_player`, `Res.font.roboto_bold`) utilizables desde cualquier plataforma del proyecto.

Reemplaza el modelo clásico de Android, donde `res/` vive únicamente dentro del módulo Android y cada plataforma (iOS, Desktop) necesitaría su propio mecanismo de recursos completamente separado.

## 2. El problema que resuelve

En un proyecto KMP sin `composeResources`, cada plataforma maneja sus recursos de forma nativa y aislada: Android usa `res/values/strings.xml`, iOS usaría `Localizable.strings` o assets del bundle, Desktop necesitaría su propio mecanismo de carga de archivos. Esto significa mantener el mismo string o el mismo ícono duplicado en 2-3 lugares distintos, con el riesgo real de que se desincronicen (un texto se corrige en Android pero no en iOS) y sin ningún chequeo de compilación que detecte un recurso faltante.

`composeResources` resuelve esto centralizando todo en `commonMain`, con generación de código type-safe: si un string no existe, es un error de compilación (`Res.string.nombre_inexistente` no compila), no un crash en runtime como pasaría con un `getString(R.string.nombre_inexistente)` mal tipeado en el modelo clásico de Android.

## 3. Ejemplo mínimo comentado

```kotlin
// Estructura de carpetas, dentro de commonMain:
// composeResources/
//   values/strings.xml       -> <string name="players_title">Jugadores</string>
//   drawable/icon_player.xml
//   font/roboto_bold.ttf

@Composable
fun PlayersScreen() {
    Column {
        Text(
            text = stringResource(Res.string.players_title), // type-safe, generado por el compilador
            fontFamily = FontFamily(Font(Res.font.roboto_bold))
        )
        Image(
            painter = painterResource(Res.drawable.icon_player),
            contentDescription = stringResource(Res.string.player_icon_description)
        )
    }
}
```

`stringResource()` y `painterResource()` son composables que leen el recurso correspondiente a la plataforma actual (por ejemplo, aplicando la localización activa del dispositivo para strings) de forma transparente — el código de UI es idéntico sin importar en qué plataforma corra.

## 4. Matriz de criterio

**`composeResources` vs recursos nativos por plataforma (`res/` de Android, assets de iOS)**
- Usar `composeResources` cuando: el recurso es genuinamente compartido entre plataformas — la gran mayoría de strings e imágenes de una app KMP entran en este caso.
- Usar recursos nativos por plataforma cuando: el recurso es específico de una plataforma por razones técnicas (por ejemplo, un ícono de notificación de Android que debe cumplir el formato específico que exige el sistema, o metadata de `Info.plist` en iOS) — esos no pasan por Compose Multiplatform en absoluto, son configuración nativa de esa plataforma puntual.
- Trade-off: centralizar todo en `commonMain` es la opción correcta por default, pero no elimina la necesidad de conocer cuándo un recurso es inherentemente nativo y no debería forzarse a compartirse.

**Strings hardcodeados en el composable vs `stringResource()`**
- Usar `stringResource(Res.string.x)` cuando: siempre, salvo texto verdaderamente interno de desarrollo (logs, nombres de debug) — cualquier texto visible al usuario final debería vivir en `composeResources`, nunca como un `String` literal dentro del composable.
- NO hardcodear cuando: el texto necesitará traducirse a otro idioma en el futuro, o simplemente por consistencia — centralizar todos los strings en un solo lugar facilita auditar el copy completo de la app sin recorrer cada archivo `.kt`.
- Trade-off: agrega un paso extra (declarar el string en el `.xml` antes de poder usarlo) comparado con tipear el texto directo — fricción mínima a cambio de mantenibilidad real.

**Nombres de recursos**
- Usar cuando: nombres descriptivos y consistentes (`players_title`, `icon_player`, no `text1`, `img_final_v2`) — el nombre generado (`Res.string.players_title`) es lo que aparece en el autocompletado del IDE, así que un mal nombre ahí se sufre en cada uso futuro.

## 5. Caso trampa

```kotlin
@Composable
fun SaveConfirmationDialog(playerName: String) {
    AlertDialog(
        onDismissRequest = {},
        text = { Text("¿Confirmás guardar a $playerName?") }, // string hardcodeado con interpolación
        confirmButton = { /* ... */ }
    )
}
```

La trampa: parece inofensivo — es solo un mensaje de confirmación, y usar `composeResources` para algo tan puntual puede sentirse como sobre-ingeniería. Pero en cuanto la app necesita soportar más de un idioma (o incluso simplemente cambiar el copy desde un lugar centralizado sin buscar en archivos `.kt` dispersos), este string queda fuera del sistema — nadie lo va a encontrar buscando en `composeResources/values/strings.xml`, y quedará hardcodeado permanentemente a menos que alguien lo detecte manualmente durante una revisión. La forma correcta es declarar un string con placeholder: `<string name="save_confirmation">¿Confirmás guardar a %1$s?</string>` y usarlo como `stringResource(Res.string.save_confirmation, playerName)` — Compose Multiplatform soporta placeholders posicionales igual que el sistema de recursos clásico de Android.

## 6. Conexión con arquitectura real

En Timbax, todo el copy visible (títulos de pantalla, labels de botones, mensajes de confirmación) y los assets visuales (íconos de jugador, logos) viven en `composeResources` dentro de `commonMain` — coherente con el objetivo general de KMP de maximizar código compartido documentado en `todosobreKMP`. Es también la razón por la cual componentes como `AddPlayerForm` (`material3_componentes_comunes.md`) nunca deberían tener un string tipeado directo en el `Text()`: el label "Nombre del jugador" de ese ejemplo, en una implementación real de Timbax, sería `stringResource(Res.string.add_player_name_label)`, manteniendo consistencia con el resto del copy de la app y dejando la puerta abierta a soporte multi-idioma sin refactors futuros.