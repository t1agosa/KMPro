# Cómo traducir un pedido de PO

## 1. Qué es

Es el proceso de convertir un pedido de producto —normalmente vago, funcional y sin vocabulario técnico— en una decisión de arquitectura concreta: qué capa toca, qué modelo cambia, qué UseCase nuevo (o existente) resuelve la regla de negocio, y qué impacto tiene en el State de la pantalla. No es "escuchar y programar" — es un paso de traducción intermedio que casi nunca se hace consciente, y que quema más tiempo cuando se salta que cuando se hace bien.

## 2. El problema que resuelve

Un PO no habla en términos de `UseCase`, `Repository` o `State` — habla en términos de comportamiento visible: "que el jugador no pueda anotar de nuevo si ya cerró la ronda", "que se vea un aviso si se queda sin conexión". Si un dev traduce eso directo a código sin pasar por un paso de análisis, dos cosas fallan seguido:

- **Se resuelve en la capa equivocada.** El caso típico: una regla de negocio ("no permitir score negativo") se termina validando en el Composable o en el ViewModel, porque ahí es donde "se ve" el problema, cuando en realidad es una invariante de dominio que debería vivir en un UseCase (o en el propio modelo) para que valga sin importar desde dónde se invoque.
- **Se sobre-construye o se sub-construye.** Sin traducir el pedido a su alcance real, es fácil crear un UseCase nuevo para algo que ya cubre uno existente, o al revés: forzar un UseCase existente a hacer algo que no le corresponde, mezclando dos responsabilidades en una sola clase.

Traducir el pedido antes de tocar código es lo que evita ambos errores — es el paso que un junior se salta y un senior hace casi automático.

## 3. Ejemplo mínimo comentado

Pedido real de PO en Timbax: *"Si un jugador llega a 100 puntos, el juego tiene que terminar solo y mostrar quién ganó."*

Traducción a arquitectura, paso por paso:

```kotlin
// 1. ¿Es una regla de negocio o de presentación?
// "Llegar a 100 puntos" es una condición sobre el dominio (Player, Score) → domain.
// "Mostrar quién ganó" es una consecuencia visible → presentation reacciona a domain.

// 2. Domain: nueva invariante, no nuevo modelo
// No hace falta un modelo "GameOver" nuevo — alcanza con una función de dominio
// que evalúe la lista de Players actual.
fun checkGameFinished(players: List<Player>, winningScore: Int = 100): Player? =
    players.firstOrNull { it.score >= winningScore }

// 3. UseCase: quién orquesta la regla
class CheckGameFinishedUseCase {
    operator fun invoke(players: List<Player>): Player? =
        checkGameFinished(players)
}
```

```kotlin
// 4. Presentation: el State se entera, no decide
data class ScoreboardState(
    val players: List<Player> = emptyList(),
    val winner: Player? = null // null = partida en curso
)

// el ViewModel llama al UseCase después de cada actualización de score,
// y el State refleja el resultado — la UI solo "lee" winner, no lo calcula
```

Notá que el pedido del PO ("termina solo y muestra quién ganó") se partió en tres decisiones técnicas distintas antes de escribir una sola línea de UI.

## 4. Matriz de criterio

| Pregunta a hacerte primero | Si la respuesta es "sí" | Si la respuesta es "no" |
|---|---|---|
| ¿La regla se cumple sin importar la pantalla desde donde se dispare? | Es dominio (UseCase o modelo) | Es presentation (lógica de esa pantalla puntual) |
| ¿Ya existe un UseCase que hace algo *casi* igual? | Evaluá extenderlo o parametrizarlo antes de crear uno nuevo | Creá uno nuevo — forzar uno existente mezcla responsabilidades |
| ¿El pedido menciona un dato que hoy no existe en ningún modelo? | Se necesita tocar `domain/model` primero | El cambio probablemente es solo de `presentation`/`ui` |
| ¿El comportamiento pedido depende de qué tan rápido pasan las cosas (ej: un timer, un debounce)? | Mirá primero si conviene resolverlo con Flow/operadores antes que con estado manual | Con State + Event alcanza, no hace falta Flow especial |

**Trade-off real:** traducir bien el pedido antes de programar cuesta minutos de análisis consciente. Saltearlo cuesta horas de refactor cuando el PO pide el siguiente cambio y la lógica quedó pegada en el lugar equivocado.

## 5. Caso trampa

PO pide: *"Que se pueda deshacer el último punto anotado, por si el jugador tocó mal."*

La respuesta obvia — y equivocada — es tratarlo como un problema de UI: "guardo el último valor en una variable `remember` y lo restauro con un botón". Eso funciona... hasta que el jugador rota la pantalla, o hasta que "deshacer" tiene que sobrevivir a que la app pase a background. El problema real es que "deshacer" es un cambio de *estado de dominio* (el score volvió a un valor anterior), no un detalle visual — necesita vivir donde vive el resto del estado del juego (ViewModel/State, potencialmente persistido), no en memoria efímera de un Composable. La trampa es que el pedido "suena" a UI porque se dispara con un botón, pero lo que hace ese botón es mutar dominio.

## 6. Conexión con arquitectura real (Timbax)

Este mismo proceso es literalmente el que definió cómo Timbax terminó separando `CheckGameFinishedUseCase` de la lógica de cálculo de puntaje: el pedido original de "que el juego avise cuando alguien gana" se tradujo primero a "¿esto es una condición de dominio?" antes de decidir en qué capa iba — evitando terminar con un `if` de fin de partida enterrado en el Composable de la pantalla de scoreboard, que es exactamente el tipo de decisión que después es carísima de deshacer.