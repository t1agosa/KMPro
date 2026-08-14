# Pipeline de Promoción

## 1. Qué es

Es el proceso mediante el cual **una misma build** avanza de forma progresiva por distintos ambientes de distribución (internal → closed/alpha → open/beta → producción) sin recompilar ni resubir un nuevo artefacto en cada etapa. La idea central: el binario que finalmente llega a producción es *exactamente* el mismo que ya fue probado en las etapas previas — solo cambia el público que lo recibe.

En Android (Google Play Console), esto se llama literalmente "**Promote release**": tomás un release que ya está en un track (ej: internal testing) y lo promovés a otro track (closed testing, open testing, production) sin volver a subir el `.aab`. En iOS (TestFlight), el equivalente es agregar el mismo build ya subido a un nuevo grupo de testers (internal → external), en vez de generar un nuevo build.

## 2. El problema que resuelve

Sin un pipeline de promoción explícito, es común caer en: *"probamos la build X en staging, pero para producción compilamos de nuevo con otra configuración"* — lo cual reintroduce exactamente el riesgo que querías evitar (una build de producción que nadie probó realmente, porque el binario que se testeó no es bit-a-bit el mismo que se publicó). El pipeline de promoción garantiza trazabilidad: sabés con certeza que el artefacto que están usando 1000 testers de closed testing es el mismo artefacto que, si se aprueba, van a recibir todos los usuarios de producción.

## 3. Ejemplo mínimo comentado

**Google Play Console — tracks disponibles y su propósito:**

```text
Internal testing   → hasta 100 testers, sin revisión de Google, disponible en minutos
Closed testing      → grupo elegido por vos (listas de email), revisión de Google la primera vez
Open testing         → testers ilimitados o con tope, cualquiera puede sumarse desde la store
Production          → todos los usuarios de Play Store, en los países configurados
```

Flujo típico: subís el `.aab` una sola vez al track `internal`. Una vez validado, usás el botón **"Promote release"** para moverlo a `closed`, después a `open`, y finalmente a `production` — sin volver a subir el archivo ni cambiar el `versionCode`.

**TestFlight (iOS) — grupos de testers:**

```text
Internal testers  → hasta 100, con cuenta en App Store Connect del equipo, sin revisión
External testers  → hasta 10.000, requieren Beta App Review (solo la primera build de cada versión)
```

Acá la "promoción" es distinta en mecánica: no herás un botón que mueva el binario, sino que **agregás el mismo build ya subido** a un nuevo grupo (de Internal a un grupo External) — Apple revisa esa build una sola vez por versión, y builds subsiguientes de la misma versión no vuelven a pasar por Beta App Review completo.

## 4. Matriz de criterio

| Escenario | Usar pipeline de promoción | NO usar / alternativa |
|---|---|---|
| Cualquier release real de una app publicada (Timbax en Play Store/App Store) | Sí, siempre — es la práctica estándar recomendada por ambas plataformas | — |
| Proyecto personal en desarrollo activo, sin usuarios reales aún | Podés saltear etapas (subir directo a `internal` y probar ahí) — no hace falta el pipeline completo hasta que haya usuarios reales | — |
| Necesitás cambiar configuración (BuildKonfig) entre testing y producción | **Cuidado**: si cambiás `BASE_URL` u otro valor de BuildKonfig entre ambientes, ya no es "la misma build" en sentido estricto — evaluá si ese cambio realmente necesita variar por ambiente o si podés testear contra el mismo backend en todas las etapas | Backend único de pruebas para toda la etapa de testing, cambiando solo al pasar a producción |
| Apps con revisión manual larga en las stores (ver `conceptos_base_branching.md`, Git Flow) | El pipeline de promoción es el motivo *real* por el que a veces conviene un ciclo de release más pausado — la promoción entre tracks lleva tiempo (revisión de Google/Apple), no es instantánea | — |

## 5. Caso trampa

**"Promoví el release de closed a production en Play Console pero algunos usuarios siguen viendo la versión vieja."**

La trampa: la promoción de release en Play Console no siempre reemplaza instantáneamente el track de destino. Si el track de producción ya tenía un rollout activo con un `versionCode` mayor o un rollout parcial en curso, Google marca la relación entre tracks como "shadowed" (un release de mayor versionCode "tapa" a otro) o "promoted" — hay reglas de fallback entre tracks que determinan qué build recibe cada usuario según en qué track está inscripto, no es un simple "reemplazar y listo". Antes de asumir que algo está roto, hay que revisar el estado de fallback de los tracks involucrados en Play Console, no solo el estado del release recién promovido.

**Trampa relacionada en iOS:** promover a testers externos no es instantáneo la primera vez — la primera build de cada versión nueva pasa por Beta App Review de Apple, que en 2026 puede tardar de horas a varios días según la cola. Subestimar ese tiempo al planificar un lanzamiento con testers externos es un error común — no es un simple "agregar al grupo y ya está disponible".

## 6. Conexión con arquitectura real (Timbax)

Para Timbax, un pipeline de promoción realista sería: build subida automáticamente al track `internal` de Play Console desde CI (ver `cicd_multi_ambiente.md`) en cada merge a `main`; una vez validada manualmente, promoción manual a `closed testing` con un grupo reducido de jugadores reales de las partidas de cartas; y solo tras esa validación, promoción a `production`. La capa `data`/BuildKonfig se mantiene constante durante todo ese pipeline (mismo backend) — lo único que cambia es el público que recibe el binario, no la configuración interna de la app, precisamente para que la promoción sea válida en el sentido estricto del término.