# merge_vs_rebase

## 1. Qué es

**Merge** y **rebase** son las dos formas de traer cambios de una rama a otra en Git. Ambos resuelven el mismo problema ("quiero los commits de la rama A también en la rama B"), pero lo hacen de forma completamente distinta:

- **Merge** crea un nuevo commit (el "merge commit") que une el historial de ambas ramas, preservando exactamente cómo pasaron las cosas en el tiempo.
- **Rebase** reescribe el historial de tu rama, tomando tus commits y reaplicándolos uno por uno sobre la punta actual de la otra rama, como si hubieras empezado a trabajar desde ahí. El resultado es un historial lineal, sin merge commits.

## 2. El problema que resuelve

Cuando dos ramas avanzan en paralelo (por ejemplo, vos trabajando en `feature/players-list` mientras `main` sigue recibiendo otros commits), en algún momento necesitás juntar ambos caminos. Sin un mecanismo explícito para esto, no habría forma de combinar el trabajo de ambas ramas manteniendo la integridad del historial — y la pregunta de fondo que separa a merge de rebase es **si te importa preservar exactamente cómo pasaron las cosas, o si preferís un historial más limpio y lineal aunque eso implique reescribir hashes de commits**.

## 3. Ejemplo mínimo comentado

```bash
# MERGE: crea un commit extra que documenta la unión de ambas ramas
git checkout main
git merge feature/players-list
# resultado: "Merge branch 'feature/players-list' into main"
# el historial queda con "forma de diamante"
```

```bash
# REBASE: reescribe tus commits como si arrancaran después
# del último commit de main (nuevos hashes)
git checkout feature/players-list
git rebase main
# resultado: historial lineal, sin merge commit
```

```bash
# flujo típico en equipo: rebase tu propia rama contra main
# ANTES de abrir el PR, para traer los últimos cambios sin ensuciar
# el historial y resolver conflictos de a poco
git checkout feature/players-list
git rebase main

# luego, el PR se mergea a main normalmente (generalmente squash, ver archivo siguiente)
```

## 4. Matriz de criterio

| Operación | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **Rebase** | Actualizar tu propia rama de feature contra `main` antes de abrir el PR — rama que nadie más bajó | La rama ya fue pusheada y **alguien más la bajó** (rama compartida) | Historial lineal y limpio, pero reescribe hashes — peligroso si otros ya tienen esos commits localmente |
| **Merge** | Integrar oficialmente un PR aprobado a la rama principal | Nunca es realmente "incorrecto" usarlo — es la opción segura por defecto | No reescribe nada (siempre seguro), pero ensucia el historial con merge commits si se abusa en ramas propias |

**Regla de oro:** nunca hagas rebase de una rama que ya compartiste con otros, salvo que todo el equipo lo sepa y coordine explícitamente.

Pregunta típica: *"¿cuándo usarías rebase y cuándo merge?"* → Rebase para mantener tu propia rama actualizada contra `main` antes de abrir PR (historial limpio, la rama es tuya, nadie más la bajó); merge para integrar oficialmente un PR aprobado a la rama principal (no reescribe historial compartido).

## 5. Caso trampa

**Situación:** Estás trabajando en `feature/players-list` junto a otro compañero que también bajó esa misma rama para ayudarte con un componente. Vos hacés `git rebase main` en tu rama local para "limpiar el historial antes del PR", y pusheás con `git push --force`.

**Por qué la respuesta obvia es incorrecta:** La regla que aprendiste ("rebasea tu rama antes del PR") asume que la rama es **tuya y de nadie más**. Acá dejó de serlo en el momento en que tu compañero la bajó. Al rebasear, reescribiste los hashes de tus commits — cuando tu compañero haga `git pull`, Git va a ver dos historiales divergentes (el suyo, basado en los hashes viejos, y el tuyo nuevo en el remoto) y va a generar conflictos confusos o, peor, un merge accidental que duplica commits. El force push encima puede pisar directamente el trabajo que tu compañero ya había subido. La señal correcta acá no es "¿es mi rama de feature?" sino "¿alguien más además de mí tiene una copia local de estos commits?" — si la respuesta es sí, se coordina con el equipo antes de rebasear, o directamente se usa merge.

## 6. Conexión con Timbax

Como Timbax hoy lo desarrollás vos solo, cualquier rama de feature es, por definición, una rama que nadie más bajó — por eso rebasear contra `main` antes de cerrar una feature es siempre seguro y te deja un historial lineal fácil de leer en el portfolio de GitHub. El caso trampa de arriba es exactamente el escenario que aparecería el día que sumes un colaborador a Timbax (o en cualquier trabajo en equipo): ahí la regla deja de ser automática y pasa a depender de si la rama sigue siendo "tuya" en términos exclusivos.