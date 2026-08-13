# github_actions_cicd

## 1. Qué es

**GitHub Actions** es el sistema de automatización de GitHub: ejecuta *workflows* (scripts definidos en YAML) en respuesta a eventos del repositorio (push, pull request, release, etc.), sin depender de que alguien corra esos pasos a mano.

**CI (Continuous Integration)** es la práctica de correr automáticamente tests y validaciones cada vez que se sube código, para detectar problemas apenas se introducen. **CD (Continuous Deployment/Delivery)** extiende esa automatización para también desplegar — subir una build a Firebase App Distribution, TestFlight, o publicar a una store.

## 2. El problema que resuelve

Sin CI, cada dev corre tests y lint "a mano" antes de subir código — lo cual depende 100% de la disciplina individual: alguien apurado, o que se olvida, sube código roto a `main` y nadie se entera hasta que otro lo baja y le explota localmente (o peor, hasta que llega a producción). Sin CD, cada release es un proceso manual repetitivo (compilar, firmar, subir) propenso a errores humanos y que no escala si el equipo crece o las releases se vuelven más frecuentes.

## 3. Ejemplo mínimo comentado

```yaml
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'
      - name: Run tests
        run: ./gradlew test
      - name: Run lint
        run: ./gradlew detekt
```

```yaml
# fragmento extra: usando un secreto para firmar/buildear con una API key,
# sin exponerla nunca en el código ni en los logs del workflow
- name: Build with API key
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: ./gradlew assembleRelease
```

Conceptos clave del archivo YAML:
- `on:` define el trigger (acá, cada PR contra `main`).
- `jobs:` cada job corre en un runner (máquina virtual) independiente, potencialmente en paralelo con otros jobs.
- `steps:` acciones secuenciales dentro de un job — `uses:` reutiliza una Action ya escrita (de GitHub o de la comunidad), `run:` ejecuta un comando de shell directo.

## 4. Matriz de criterio

| Concepto | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **CI (tests + lint automáticos)** | Siempre que haya más de una persona tocando el repo, o incluso solo (para no confiar en la memoria de correr tests a mano) | — (prácticamente no hay escenario donde CI no sume valor, salvo prototipos totalmente descartables) | Tiempo de espera en cada PR (minutos), a cambio de detectar problemas antes de que lleguen a `main` |
| **CD hacia stores/distribución interna** | Releases frecuentes, o builds de prueba que un equipo (QA, testers) necesita bajar seguido | Proyecto en etapa muy temprana donde cada build manual sirve para revisar el proceso en sí | Ahorra trabajo repetitivo y reduce error humano, pero exige configurar correctamente firma de builds y secretos — mal configurado, puede publicar algo no revisado |
| **Branch protection rules** | Cualquier repo donde `main` representa código en producción o cerca de eso | Repos personales de prueba/aprendizaje sin consecuencias reales de romper `main` | Fuerza que el flujo de PR + CI sea obligatorio, no una convención que alguien puede saltearse por apuro |

Pregunta típica: *"¿cómo asegurarías que nadie mergea código roto a main?"* → Configurando branch protection rules que exijan CI en verde y aprobación de review antes de habilitar el botón de merge — no confiando en que cada dev corra los tests a mano antes de mergear.

## 5. Caso trampa

**Situación:** Configurás un workflow de CI que corre tests y lint en cada PR, y lo das por "resuelto": ya no hace falta branch protection porque "el CI ya corre y si falla, se ve en rojo en el PR — el equipo no va a mergear algo en rojo".

**Por qué la respuesta obvia es incorrecta:** Que el CI corra y muestre el resultado no impide que alguien mergee igual. Sin **branch protection rules** configuradas explícitamente, el botón de "Merge" en GitHub sigue habilitado aunque el check de CI esté en rojo — queda librado a que la persona *elija* no apretarlo, exactamente el mismo problema de depender de la disciplina individual que el CI se supone que resuelve. El CI por sí solo es información; branch protection es lo que convierte esa información en una regla imposible de saltear. Es un error común pensar que tener el pipeline configurado ya cierra el problema, cuando falta el segundo paso que lo hace realmente obligatorio.

## 6. Conexión con Timbax

Timbax se beneficia de CI incluso trabajando solo: un workflow que corre `./gradlew test` en cada PR (o incluso en cada push a una rama de feature) te avisa si rompiste algo antes de que vos mismo lo notes recién al abrir la app — especialmente útil en un proyecto KMP donde un cambio en `commonMain` puede romper silenciosamente el build de una plataforma que no estás mirando en ese momento (por ejemplo, tocar algo y romper el target iOS sin darte cuenta porque estás compilando y probando solo en Android). El paso de CD (subir una build a Firebase App Distribution automáticamente al mergear a `main`) es el siguiente escalón natural una vez que Timbax tenga testers externos probando versiones antes de publicarlas en la store.