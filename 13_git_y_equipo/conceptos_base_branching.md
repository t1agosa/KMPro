# conceptos_base_branching

## 1. Qué es

Git es un sistema de control de versiones distribuido: cada desarrollador tiene una copia completa del historial del repositorio en su máquina, no solo el archivo actual. Sobre esa base viven dos conceptos centrales: el **commit** (una fotografía del código en un momento dado, con mensaje y hash único que lo encadena al anterior) y el **branch** (un puntero móvil a un commit, no una copia del código).

Sobre esos dos conceptos se paran las **estrategias de branching**: las reglas que un equipo define para organizar cómo y cuándo se crean, viven y mueren esas ramas. Las dos más relevantes hoy son **Trunk-Based Development** (ramas cortas que vuelven rápido a `main`) y **Git Flow** (ramas de larga duración con roles fijos: `main`, `develop`, `feature/*`, `release/*`, `hotfix/*`).

## 2. El problema que resuelve

Sin control de versiones distribuido, el historial de cambios vive en un solo lugar (o no vive en ningún lado), y trabajar en paralelo sobre el mismo código sin pisarse es prácticamente imposible. Sin branches, cualquier cambio en progreso convive con el código estable en el mismo lugar — no hay forma de experimentar, romper algo a mitad de camino, o tener a dos personas trabajando en features distintas sin bloquearse mutuamente.

Y sin una estrategia de branching acordada, cada dev decide por su cuenta cuánto tiempo vive una rama y cuándo vuelve a integrarse — lo que en la práctica genera ramas que viven semanas, divergen cada vez más de `main`, y terminan en un merge doloroso de cientos de líneas en conflicto. Cuanto más tiempo vive una rama sin mergear, más "drift" acumula.

## 3. Ejemplo mínimo comentado

```bash
# crear y cambiar a una rama nueva, partiendo del estado actual de main
git checkout -b feature/players-list

# ... trabajás, hacés commits chicos ...

# volver a main
git checkout main

# listar ramas locales (la actual aparece marcada con *)
git branch

# borrar una rama ya mergeada (git avisa si todavía tiene cambios sin mergear)
git branch -d feature/players-list
```

```bash
# ejemplo de branch en Git Flow: un hotfix urgente
# sale directo de main (no de develop), porque no puede esperar
# al próximo ciclo de release
git checkout main
git checkout -b hotfix/fix-negative-score

# ... arreglo puntual ...

# vuelve a AMBOS: main (para producción ya) y develop (para no perderlo)
```

## 4. Matriz de criterio

| Estrategia | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **Trunk-Based** | Equipo chico/mediano, CI/CD real, releases frecuentes (varias veces por semana o por día) | El proyecto tiene ciclos de aprobación manual largos (ej: revisión de una store que tarda días) | Exige disciplina: ramas cortas de verdad (horas, no semanas) y PRs chicos. Sin esa disciplina, degenera en el mismo drift que se quería evitar |
| **Git Flow** | Releases espaciadas y controladas, versiones LTS, necesidad de mantener varias versiones en producción a la vez | Equipos chicos con releases frecuentes — la estructura de ramas (`develop`, `release/*`, `hotfix/*`) agrega complejidad que no se paga sola | Da más estructura y trazabilidad por versión, a costa de más ceremonia y más superficie para que el código diverja entre `main` y `develop` |

Pregunta típica de entrevista: *"¿qué estrategia usarías en un equipo chico con releases frecuentes?"* → Trunk-based, porque Git Flow agrega complejidad (múltiples ramas de larga vida) que solo se justifica en proyectos con ciclos de release muy espaciados.

## 5. Caso trampa

**Situación:** Tu equipo es chico (3 personas), hace releases cada 2 semanas a producción, y alguien propone adoptar Git Flow completo "porque es lo más profesional y lo usan las empresas grandes".

**Por qué la respuesta obvia es incorrecta:** Git Flow no es "más profesional" en abstracto — es una solución a un problema específico (coordinar múltiples versiones en producción simultáneamente, con ciclos de release largos y espaciados). Un equipo de 3 personas con releases cada 2 semanas no tiene ese problema. Adoptar Git Flow ahí significa mantener `develop` sincronizado con `main`, decidir cuándo abrir `release/*`, y gestionar `hotfix/*` en paralelo — ceremonia que no resuelve ningún dolor real del equipo, solo lo agrega. La señal correcta no es "qué estrategia es más rigurosa", sino "qué problema tengo hoy" — y la respuesta más senior acá es trunk-based con PRs chicos y frecuentes.

## 6. Conexión con Timbax

Timbax, como proyecto de una sola persona con releases que vos mismo controlás, es el caso de uso de libro para trunk-based: ramas cortas por feature (`feature/agregar-chinchon`, `fix/calculo-negativo`) que salen de `main` y vuelven rápido, sin necesidad de un `develop` intermedio ni de coordinar hotfixes en paralelo con una release en curso. Si en algún momento Timbax tuviera que mantener, por ejemplo, una versión "free" y una "premium" en producción al mismo tiempo con ciclos de vida distintos, ahí recién Git Flow (o una variante) empezaría a pagarse solo.

---

## Anexo: git stash

Herramienta relacionada pero distinta a branching: guarda los cambios sin commitear (working directory + staging area) en una pila temporal local, dejando el working directory limpio. Nunca se pushea — vive solo en el `.git` local del desarrollador. Se usa para cambiar de contexto rápido (checkout urgente a otra rama, pull) sin commitear código a medio terminar ni perderlo.

```bash
# guarda tus cambios sin commitear y limpia el working directory
git stash

# ... cambiás de rama, hacés otra cosa ...

# recuperás los cambios guardados (el más reciente del stack)
git stash pop

# ver la lista de stashes guardados (podés tener varios)
git stash list

# aplicar sin borrar del stack (pop = apply + drop)
git stash apply
```

No sustituye a un commit ni a un branch — es un guardado temporal y descartable, útil para interrupciones cortas, no para trabajo que necesita historial o que vas a compartir con nadie.