# squash_y_conflictos

## 1. Qué es

**Squash** es una operación que toma todos los commits de una rama y los combina en uno solo antes de mergearlos a la rama principal. Es la tercera opción además de merge y rebase directos, y en GitHub aparece típicamente como "Squash and merge" al cerrar un Pull Request.

**Conflicto de merge** es lo que ocurre cuando dos ramas modificaron las mismas líneas de un archivo de forma distinta, y Git no puede decidir automáticamente cuál versión es la correcta — pasa tanto al usar `merge` como al usar `rebase`.

## 2. El problema que resuelve

**Squash** resuelve el ruido de commits intermedios: durante el desarrollo de una feature hacés commits tipo `wip`, `fix typo`, `arreglo real`, `ahora sí anda` — commits que solo tenían sentido mientras trabajabas, pero que en el historial de `main` no aportan nada, solo ensucian. Squash colapsa todo eso en un commit limpio y descriptivo por feature completa.

**Los conflictos** no son un "error" de Git — son la consecuencia inevitable de que dos personas (o vos mismo en dos ramas) toquen la misma línea de forma distinta. Git no tiene forma de saber cuál versión es la "correcta" en términos de intención de negocio, así que te lo devuelve para que decidas vos.

## 3. Ejemplo mínimo comentado

```bash
# squash manual: combina los últimos 3 commits en uno solo,
# antes de abrir el PR (alternativa a dejar que GitHub lo haga al mergear)
git rebase -i HEAD~3
# se abre un editor con los 3 commits listados como "pick";
# cambiás "pick" por "squash" (o "s") en los que querés combinar
# con el commit anterior
```

```bash
# conflicto de merge: Git marca directamente el archivo afectado
```

```kotlin
<<<<<<< HEAD
val score = player.score + 10
=======
val score = player.score * 1.1
>>>>>>> feature/bonus-multiplier
```

```bash
# resolución: editar el archivo a mano, decidiendo qué código queda
# (combinando si hace falta), y borrar los marcadores
# <<<<<<<, =======, >>>>>>>

git add ScoreCalculator.kt

# si el conflicto surgió durante un rebase:
git rebase --continue

# si el conflicto surgió durante un merge:
git commit
```

## 4. Matriz de criterio

| Concepto | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **Squash** | Cerrar un PR con commits intermedios ruidosos (wip, fixes de typo) que no aportan valor al historial de `main` | La feature es grande y cada commit intermedio SÍ documenta una decisión útil de revisar por separado (poco común, pero pasa en refactors complejos) | Historial de `main` limpio (un commit = una feature), pero perdés la granularidad de los commits intermedios si alguna vez necesitás revisar el paso a paso |
| **Resolución de conflictos** | Siempre que Git no pueda auto-mergear — no hay forma de evitarlo, solo de manejarlo bien | — (no es opcional, es inevitable en trabajo paralelo) | El costo es tiempo y atención: entender la intención de ambos cambios, no solo "pegar los dos" a ciegas |

Pregunta típica: *"¿por qué muchos equipos usan squash merge por default en GitHub?"* → Porque mantiene el historial de `main` limpio y legible (un commit = una feature/fix completo), sin el ruido de los commits intermedios que solo tenían sentido mientras trabajabas.

## 5. Caso trampa

**Situación:** Tenés un conflicto de merge en `ScoreCalculator.kt`. Ves los dos bloques (`<<<<<<<` y `>>>>>>>`) y, para resolverlo rápido, dejás **ambas** líneas de código, una debajo de la otra, pensando "así no pierdo el trabajo de nadie".

**Por qué la respuesta obvia es incorrecta:** Un conflicto no es un problema de "falta contenido", es un problema de **decisión contradictoria**. En el ejemplo del punto 3, una rama calcula `score + 10` y la otra `score * 1.1` — son dos reglas de negocio distintas e incompatibles para lo mismo. Dejar las dos líneas no es "no perder nada": es dejar código que probablemente ni compile bien, o que sí compile pero ejecute dos operaciones donde el negocio esperaba una sola, produciendo un resultado que nadie pidió. Resolver un conflicto bien significa entender **por qué** cada rama hizo ese cambio (leer el commit message, el contexto del PR) y decidir cuál regla es la vigente — o si hay que combinarlas de una forma que tenga sentido real (por ejemplo, aplicar el bonus multiplicador y después sumar el fijo), no pegar ambas líneas literalmente una arriba de la otra.

## 6. Conexión con Timbax

En Timbax, trabajando solo, los conflictos de merge son raros pero no imposibles — pasan, por ejemplo, si tenés dos ramas de feature abiertas en paralelo (una tocando `ScoreCalculator.kt` para Chinchón, otra tocando el mismo archivo para Truco) y las mergeás en desorden. El squash, en cambio, lo usás todo el tiempo sin pensarlo: cada PR tuyo en GitHub, aunque tenga 8 commits de `wip` mientras probabas la lógica de puntaje, termina en `main` como un solo commit `feat: agregar cálculo de puntaje bonus en Chinchón` — coherente con la convención de conventional commits que ya usás en el resto del proyecto.