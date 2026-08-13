# pull_requests_code_review

## 1. Qué es

Un **Pull Request** (PR, o Merge Request en GitLab) es una propuesta de cambio: "quiero traer los commits de mi rama `feature/X` a `main`", abierta para que el equipo la revise antes de aceptarla. No es un concepto de Git en sí — Git no tiene PRs — es una feature construida por la plataforma (GitHub, GitLab, Bitbucket) por encima de Git.

**Code review** es la práctica central del flujo de PR: compañeros leen el diff, comentan, piden cambios, y aprueban antes de que el código llegue a `main`.

## 2. El problema que resuelve

Sin PR, cualquiera podría pushear directo a `main` sin que nadie más vea el cambio antes de que llegue a producción — un solo error de un dev termina en `main` sin filtro. Sin code review, aunque exista el paso del PR como formalidad, se pierde el objetivo real: **detectar bugs es solo una parte**; la otra, igual de importante, es compartir contexto de por qué se tomó una decisión de diseño, mantener consistencia de estilo en el equipo, y evitar el "bus factor" — que una sola persona sea la única que entiende cierta parte del sistema.

## 3. Ejemplo mínimo comentado

```bash
# 1. crear rama desde main
git checkout -b feature/players-list

# 2. commits chicos y descriptivos mientras desarrollás
git commit -m "feat: agregar modelo Player y repository contract"
git commit -m "feat: implementar PlayersViewModel con estado inicial"

# 3. pushear la rama al remoto
git push origin feature/players-list

# 4. abrir el PR en GitHub (vía UI), con descripción de qué cambia y por qué
```

```kotlin
// Ejemplo de lo que un reviewer buscaría en el diff de este PR:
// - ¿El ViewModel realmente no tiene lógica de negocio filtrada?
class PlayersViewModel(
    private val getPlayersUseCase: GetPlayersUseCase // OK: delega al UseCase
) {
    // señal de alerta para un reviewer: esto NO debería estar acá
    // fun calculateBonus(score: Int) = score * 1.1  // lógica de negocio en presentation
}
```

```bash
# 5. una vez aprobado y con CI en verde, se mergea (usualmente squash) a main
# 6. se borra la rama
git branch -d feature/players-list
git push origin --delete feature/players-list
```

## 4. Matriz de criterio

| Aspecto | Buena práctica | Anti-patrón | Trade-off real |
|---|---|---|---|
| **Tamaño del PR** | Chico y de alcance acotado (una feature/fix concreto) | PR gigante que toca 15 archivos y mezcla 3 cosas distintas | PRs chicos se revisan mejor y rápido, pero exigen dividir el trabajo en pasos más pequeños — más disciplina, no menos trabajo total |
| **Qué buscar en review** | Que el cambio haga lo que dice, que no rompa la arquitectura (capas, dependencias), que tenga tests, legibilidad | Solo mirar si "compila" o si "se ve bien" superficialmente | Un review a fondo tarda más, pero un review superficial deja pasar bugs de diseño que después cuestan mucho más arreglar |
| **Aprobación** | Al menos 1 aprobación de otra persona antes de mergear (reforzado con branch protection rules) | Auto-mergear el propio PR sin revisión de nadie | Ganás velocidad a corto plazo, perdés la detección temprana y el "segundo par de ojos" que evita bugs y comparte contexto |

Pregunta típica: *"¿qué buscarías al revisar un PR de un compañero?"* → Que el cambio haga lo que dice que hace, que no rompa arquitectura existente (capas, dependencias), que tenga tests para lógica nueva, legibilidad del código, y que el alcance del PR sea razonable — PRs gigantes son difíciles de revisar bien, mejor varios PRs chicos que uno enorme.

## 5. Caso trampa

**Situación:** Sos el único senior del equipo y te llega un PR de un dev junior. El código funciona, pasa los tests, y el CI está en verde. Lo apruebas rápido porque "si compila y pasa los tests, está bien" — total, para eso están los tests.

**Por qué la respuesta obvia es incorrecta:** Que el código **funcione** y que el código esté **bien** no son lo mismo. Los tests verifican comportamiento esperado, pero no dicen nada sobre si el `ViewModel` tiene lógica de negocio que debería estar en un `UseCase`, si el nombre de una función es confuso para el resto del equipo, o si esa misma lógica ya existe en otro lado del código y ahora hay dos formas distintas de calcular lo mismo. Aprobar solo en base a "compila + tests en verde" reduce el code review a lo que el CI ya te garantiza automáticamente — en ese caso, el review humano no está aportando nada que el pipeline no aportara solo, y el objetivo real del review (mantener arquitectura consistente, transferir conocimiento, prevenir deuda técnica) queda sin cubrir.

## 6. Conexión con Timbax

En Timbax, al trabajar solo, no hay un compañero que abra el PR de code review — pero el flujo de PR sigue teniendo valor igual: abrir el PR contra vos mismo (o simplemente revisar el diff completo antes de mergear a `main`) te obliga a mirar tu propio cambio con la misma distancia crítica que le pedirías a un reviewer — "¿esta lógica está en la capa correcta?", "¿el nombre de esta función es claro sin el contexto que tengo yo ahora en la cabeza?". Es, en cierto sentido, un ensayo del hábito que vas a necesitar el día que trabajes en equipo, donde alguien más va a hacerte exactamente esas preguntas sobre tu código.