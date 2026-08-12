# theming_material3.md

## 1. Qué es

El theming en Material3 es el mecanismo por el cual toda la app comparte una única fuente de verdad para colores, tipografía y formas: un `ColorScheme`, una `Typography` y un `Shapes`, agrupados dentro de un `MaterialTheme` que envuelve toda la app desde la raíz (usualmente en el `App()` composable). Cualquier composable hijo accede a esos valores vía `MaterialTheme.colorScheme.primary`, `MaterialTheme.typography.bodyLarge`, etc., en vez de hardcodear colores o tamaños de fuente sueltos.

## 2. El problema que resuelve

Sin un theme centralizado, cada composable definiría sus propios colores y tamaños de texto de forma independiente (`Color(0xFF6200EE)` repetido en 30 archivos distintos). El resultado: inconsistencia visual, y un cambio de paleta (por ejemplo, para soportar modo oscuro) obligaría a tocar cada composable uno por uno.

`MaterialTheme` resuelve esto centralizando la paleta y la tipografía en un solo lugar. Cambiar el `ColorScheme` que se le pasa a `MaterialTheme` en la raíz de la app propaga el cambio a *toda* la UI automáticamente, porque cada composable lee `MaterialTheme.colorScheme.primary` en vez de un valor fijo — el mismo mecanismo de propagación implícita que documentamos en `compositionlocal.md` (de hecho, `MaterialTheme` está construido internamente sobre `CompositionLocal`).

## 3. Ejemplo mínimo comentado

```kotlin
private val TimbaxLightColors = lightColorScheme(
    primary = Color(0xFF2E7D32),
    onPrimary = Color.White,
    secondary = Color(0xFFFFA000)
)

private val TimbaxDarkColors = darkColorScheme(
    primary = Color(0xFF81C784),
    onPrimary = Color.Black,
    secondary = Color(0xFFFFCA28)
)

@Composable
fun TimbaxTheme(
    useDarkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (useDarkTheme) TimbaxDarkColors else TimbaxLightColors

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(), // valores default de M3, o customizados
        content = content
    )
}

// Uso en cualquier composable de la app, sin importar nada extra:
@Composable
fun ScoreLabel(score: Int) {
    Text(
        text = "$score",
        color = MaterialTheme.colorScheme.primary, // nunca Color(0xFF...) hardcodeado
        style = MaterialTheme.typography.headlineMedium
    )
}
```

`isSystemInDarkTheme()` lee la preferencia del sistema operativo automáticamente; `TimbaxTheme` decide qué `ColorScheme` aplicar según eso, y todo lo que esté dentro de `content` accede al resultado vía `MaterialTheme.colorScheme` sin saber si está en modo claro u oscuro.

## 4. Matriz de criterio

**`MaterialTheme.colorScheme.X` vs `Color(0xFF...)` hardcodeado**
- Usar `MaterialTheme.colorScheme` cuando: siempre, salvo un color verdaderamente fijo e independiente del tema (raro — ej: el color exacto de un logo de marca externa que no debe variar).
- NO hardcodear un color cuando: ese color debería adaptarse a modo claro/oscuro o a un cambio futuro de paleta — hardcodear rompe silenciosamente el modo oscuro (un texto oscuro sobre fondo oscuro, por ejemplo, sin ningún error de compilación).
- Trade-off: ninguno real en contra de usar el theme — es más código la primera vez (definir la paleta), pero es la única forma de que dark mode "simplemente funcione" en toda la app.

**Colores semánticos (`primary`/`onPrimary`, `surface`/`onSurface`, etc.)**
- Usar el par `X`/`onX` cuando: necesitás garantizar contraste — `onPrimary` siempre es el color de texto/ícono legible *sobre* `primary`, calculado para cumplir accesibilidad, sin que vos elijas manualmente "blanco o negro" cada vez.
- NO usar cuando: elegís un color "a ojo" para texto sobre un fondo de color — eso es exactamente el tipo de decisión que el sistema `X`/`onX` reemplaza, evitando bugs de contraste (texto ilegible) en modo oscuro.
- Trade-off: requiere pensar en pares desde el diseño, no en colores sueltos — más disciplina inicial, pero elimina una clase entera de bugs de accesibilidad.

**Customizar `Typography` vs usar el default de M3**
- Customizar cuando: la marca/identidad visual de la app requiere una tipografía propia (fuente custom, tamaños específicos).
- Usar el default cuando: es una app personal, MVP, o donde la identidad tipográfica no es prioridad — el default de M3 ya está pensado con buena legibilidad y jerarquía.
- Trade-off: customizar tipografía es una única vez al principio del proyecto — hacerlo tarde implica revisar cada `Text` que asumió un `style` del default.

## 5. Caso trampa

```kotlin
@Composable
fun WarningBanner(message: String) {
    Box(modifier = Modifier.background(Color(0xFFFFF3CD))) { // amarillo hardcodeado
        Text(text = message, color = Color.Black) // negro hardcodeado
    }
}
```

La trampa: en modo claro esto se ve perfecto — fondo amarillo pálido, texto negro legible. Pero como ninguno de los dos colores viene de `MaterialTheme.colorScheme`, cuando el usuario activa modo oscuro, este banner se queda exactamente igual (amarillo claro, texto negro) mientras el resto de la app cambia a paleta oscura — quedando como un parche visual completamente fuera de tono, y potencialmente con contraste incorrecto si el resto del sistema de colores fue pensado para fondos oscuros. El bug no es "se rompe": es peor, "se ve raro mezclado" — algo que un ojo entrenado nota pero que no genera ningún error para debuggear. La regla es usar un color semántico del `ColorScheme` (por ejemplo, `errorContainer`/`onErrorContainer` para un warning), que ya viene resuelto para ambos modos.

## 6. Conexión con arquitectura real

En Timbax, `TimbaxTheme` envuelve la función raíz `App()` una sola vez, y desde ahí cada `Screen` (`PlayersScreen`, `AddPlayerForm` de `material3_componentes_comunes.md`) consume `MaterialTheme.colorScheme` sin necesitar recibirlo como parámetro — el mismo mecanismo de propagación implícita de `CompositionLocal` que se documenta en `compositionlocal.md`, aplicado al caso de uso más común y "seguro" de ese mecanismo (tema visual, no lógica de negocio ni navegación). Es, además, la razón por la cual ningún composable de UI en Timbax debería tener un `Color(0xFF...)` suelto: si aparece uno en una revisión de PR, es señal de que se está esquivando el theme en vez de usarlo.