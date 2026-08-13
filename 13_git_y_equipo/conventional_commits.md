# conventional_commits

## 1. Qué es

**Conventional Commits** es una convención de formato para mensajes de commit, donde cada mensaje empieza con un prefijo que indica el tipo de cambio: `feat`, `fix`, `refactor`, `test`, `chore`, `docs`, entre otros. No es una herramienta ni una librería — es un acuerdo de formato que cualquiera puede adoptar escribiendo los mensajes de cierta forma.

```
feat: agregar pantalla de scoreboard
fix: corregir cálculo de puntaje negativo
refactor: extraer mapper de PlayerDto a función separada
test: agregar tests para SaveScoreUseCase
chore: actualizar versión de Ktor
docs: documentar arquitectura del módulo shared
```

## 2. El problema que resuelve

Sin una convención, el historial de commits de un proyecto termina siendo una lista de frases libres sin estructura (`arreglos varios`, `cambios`, `ahora sí`) — imposible de escanear rápido para entender qué pasó entre dos versiones, y completamente inútil para automatizar nada. Con Conventional Commits, el tipo de cambio queda codificado en el propio mensaje, lo que habilita dos cosas que serían manuales (y tediosas) sin la convención: generar un **changelog automático** agrupado por tipo, y que herramientas de **versionado semántico** (como `semantic-release`) decidan si el próximo release es major, minor o patch mirando los prefijos de los commits desde el último release.

## 3. Ejemplo mínimo comentado

```bash
# feat: nueva funcionalidad visible para el usuario
git commit -m "feat: agregar modo Truco a la selección de juegos"

# fix: corrección de un bug
git commit -m "fix: corregir cálculo de puntaje negativo en Chinchón"

# refactor: cambio de estructura interna, SIN cambiar comportamiento
git commit -m "refactor: extraer PlayerMapper a función de extensión separada"

# test: agregar o modificar tests, sin tocar código de producción
git commit -m "test: agregar tests de SaveScoreUseCase con FakeRepository"

# chore: tareas de mantenimiento que no afectan al usuario final
git commit -m "chore: actualizar Ktor a 3.0.1"

# docs: documentación
git commit -m "docs: agregar merge_vs_rebase.md (13_git_y_equipo)"
```

```bash
# ejemplo de por qué importa la distinción feat vs fix para versionado semántico:
# si desde el último release solo hubo "fix:", el próximo release es un PATCH (1.2.0 -> 1.2.1)
# si hubo al menos un "feat:", el próximo release es un MINOR (1.2.0 -> 1.3.0)
# un "BREAKING CHANGE:" en el body del commit fuerza un MAJOR (1.2.0 -> 2.0.0)
```

## 4. Matriz de criterio

| Prefijo | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| `feat` | El cambio agrega funcionalidad nueva visible para el usuario/negocio | Es un cambio interno que el usuario final nunca nota | Dispara un MINOR en versionado semántico automático — usarlo mal infla versiones sin motivo |
| `fix` | Corrige un comportamiento roto/incorrecto existente | Es una mejora nueva disfrazada de "arreglo" | Dispara un PATCH — clasificar mal un `feat` como `fix` esconde funcionalidad nueva del changelog |
| `refactor` | Cambia estructura/organización del código, comportamiento idéntico | El cambio sí altera comportamiento aunque sea "internamente" | No dispara ningún cambio de versión semántica por sí solo — es la señal correcta para decir "esto es seguro, no cambia nada visible" |
| `chore` | Tareas de mantenimiento (deps, configs) sin impacto directo en funcionalidad | Es en realidad un `fix` o `feat` disfrazado de tarea menor | Igual que `refactor`, no dispara versión — usarlo para esconder un cambio real rompe la trazabilidad |

Pregunta típica: *"¿por qué seguir una convención de commits?"* → Permite automatizar versionado semántico (herramientas como `semantic-release` deciden si el próximo release es major/minor/patch según los prefijos de los commits desde el último release) y generar changelogs sin trabajo manual.

## 5. Caso trampa

**Situación:** Arreglaste un bug donde el puntaje se calculaba mal en Chinchón, pero de paso — mientras estabas ahí — también reorganizaste tres funciones del archivo en un solo commit. Escribís `fix: corregir cálculo de puntaje y reorganizar ScoreCalculator`.

**Por qué la respuesta obvia es incorrecta:** El mensaje mezcla dos tipos de cambio distintos en un solo commit con un solo prefijo. Esto rompe la utilidad real de la convención: si alguien necesita revertir específicamente el `fix` (por ejemplo, porque introdujo una regresión), no puede — el revert se lleva puesta también la reorganización, que no tenía nada que ver con el bug. Además, el changelog automático va a listar esto como un solo `fix`, ocultando que también hubo un refactor — información que alguien evaluando qué cambió entre dos versiones pierde sin necesidad. La práctica correcta es un commit por intención: `fix: corregir cálculo de puntaje negativo en Chinchón` primero, y `refactor: reorganizar funciones de ScoreCalculator` después, aunque ambos terminen en el mismo PR — son dos cambios con propósitos distintos, y el historial debería poder separarlos.

## 6. Conexión con Timbax

Este es, de los siete archivos del package, el que ya venís aplicando de forma más consistente y directa en Timbax: cada archivo `.md` de este mismo repo `todosobreKMP` se commitea con el prefijo `docs:` (`docs: agregar merge_vs_rebase.md`), y el resto del código de Timbax sigue la misma lógica — `feat:` para una carta nueva soportada, `fix:` para un cálculo de puntaje corregido, `refactor:` cuando movés lógica de un `ViewModel` a un `UseCase` sin cambiar el resultado visible. Es, en la práctica, la convención que ya estás usando para cerrar cada package de este mismo proyecto de documentación.