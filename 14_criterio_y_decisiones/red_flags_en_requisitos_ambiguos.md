# Red flags en requisitos ambiguos

## 1. Qué es

Es el conjunto de señales que indican que un requisito, tal como llegó del PO, todavía no está listo para convertirse en arquitectura — porque esconde una ambigüedad que, si no se resuelve antes de programar, se termina resolviendo *por accidente* en el código, con la primera interpretación que se le ocurrió al dev en el momento. Detectar estas señales es tan parte del trabajo técnico como escribir el código en sí.

## 2. El problema que resuelve

Un requisito ambiguo no explota en el momento en que se escribe — explota semanas después, cuando el comportamiento "obvio" que se implementó no era el que el PO tenía en mente, o cuando aparece un caso borde que el requisito original nunca contempló porque nadie lo preguntó. La causa raíz casi siempre es la misma: el dev interpretó el requisito en vez de chequear la ambigüedad con el PO *antes* de programar. Tener un catálogo de red flags conocidas convierte esto de "me doy cuenta cuando ya es tarde" a "lo detecto en el momento en que leo el pedido".

## 3. Ejemplo mínimo comentado

Pedido real: *"Que se pueda editar el nombre de un jugador después de creado."*

```kotlin
// Red flag: el requisito no dice qué pasa con datos relacionados
// que ya referencian a ese jugador por nombre (no por id).

// Interpretación A (la que un dev suele asumir sin preguntar):
// el nombre es solo un campo más, se actualiza y listo.
data class Player(val id: String, val name: String, val score: Int)

fun renamePlayer(player: Player, newName: String): Player =
    player.copy(name = newName)

// Pregunta que el requisito NO contesta:
// ¿el historial de partidas ya jugadas debe mostrar el nombre VIEJO
// (foto del momento) o el nombre NUEVO (siempre el actual)?
// Esto define si el historial debe guardar el nombre como snapshot
// o resolverlo por id en tiempo de lectura — son dos diseños de
// modelo totalmente distintos, y el requisito original no lo dice.
```

La red flag acá no es el código — es que el requisito "editar nombre" suena trivial pero esconde una decisión de modelado (snapshot vs. referencia) que cambia el diseño de dominio entero si se responde mal.

## 4. Matriz de criterio

| Red flag en el requisito | Por qué es peligrosa | Qué preguntar antes de programar |
|---|---|---|
| Usa "y/o" sin aclarar cuál de los dos casos | El comportamiento real depende de una condición que nadie definió | "¿Es un Y o un O? ¿Qué pasa si se cumplen ambas condiciones a la vez?" |
| Describe el resultado pero no el caso borde (vacío, error, offline) | Los estados de excepción se terminan improvisando en el código | "¿Qué se muestra si la lista está vacía / falla la red / el usuario cancela a mitad de camino?" |
| Usa una palabra "obvia" que en realidad tiene más de un significado razonable (ej: "eliminar", "cancelar", "reiniciar") | Cada significado implica un diseño distinto (soft delete vs. hard delete, por ejemplo) | "Cuando decís 'eliminar', ¿se borra para siempre o se puede recuperar?" |
| No aclara si el cambio afecta datos ya existentes o solo los nuevos | Migraciones de datos o inconsistencias retroactivas no contempladas | "¿Esto aplica también a lo que ya está guardado, o solo hacia adelante?" |
| Pide "que sea como en [otra app]" sin especificar qué parte exactamente | Cada app resuelve el mismo problema de formas distintas — asumir cuál se referencia es adivinar | "¿Qué parte puntual de esa app querés replicar? ¿El flujo completo o solo un detalle visual?" |

**Trade-off real:** preguntar antes de programar cuesta una interrupción de 2 minutos en una conversación con el PO. No preguntar y adivinar mal cuesta un ida-y-vuelta de QA, un fix, y —peor— erosiona la confianza del PO en que el equipo técnico entiende lo que se pide.

## 5. Caso trampa

PO pide: *"Que el jugador con menos puntos pague la ronda."* Parece un requisito cerrado y sin ambigüedad — hay un ganador claro por comparación numérica. La trampa está en el caso de empate, que el requisito ni siquiera menciona porque el PO probablemente no lo pensó al escribirlo. Un dev apurado programa `players.minBy { it.score }`, que en un empate devuelve *cualquiera* de los empatados de forma determinística pero arbitraria (el primero que encuentre según el orden interno de la lista) — no porque haya una regla de negocio detrás, sino porque así funciona `minBy` cuando hay múltiples mínimos. El bug no está en el código, está en haber programado una decisión de producto (qué pasa en un empate) sin haberla hecho explícita primero. La señal de alarma no era el requisito en sí — era notar que "el jugador con menos puntos" presupone que existe un único mínimo, algo que el dominio del juego no garantiza.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, justamente el cálculo de "quién ganó" o "quién debe" (en juegos con acumulación de puntaje) pasó por esta misma revisión: antes de escribir el `UseCase` correspondiente, hubo que confirmar explícitamente con la regla de negocio real del juego qué pasa en un empate, en vez de dejar que `minBy`/`maxBy` decidiera algo que en realidad era una decisión de producto sin resolver — el mismo patrón que este archivo describe en abstracto.