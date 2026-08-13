# secretos_gitignore

## 1. Qué es

**`.gitignore`** es un archivo que le dice a Git qué **no** trackear: carpetas de build, configuración local del IDE, archivos de credenciales. En KMP típicamente ignora `build/`, `.gradle/`, `.idea/`, `*.xcuserstate`, `local.properties`.

**Manejo de secretos** es la práctica de nunca commitear API keys ni credenciales al repo — ni siquiera en un archivo gitignoreado que alguien podría subir por error — y en cambio manejarlas vía **GitHub Secrets** (variables encriptadas inyectadas al workflow en runtime) para CI, y vía **BuildKonfig** o variables de entorno locales para desarrollo.

## 2. El problema que resuelve

Sin `.gitignore`, cada `git status` se llena de ruido (carpetas de build regenerables, configs del IDE que cambian por persona) y es fácil commitear por accidente algo que no debería estar versionado. Sin manejo cuidadoso de secretos, una API key terminaría literalmente en el código fuente — visible para cualquiera con acceso al repo (y si el repo es público, para cualquiera en internet, incluso bots que escanean GitHub constantemente buscando keys filtradas).

## 3. Ejemplo mínimo comentado

```bash
# .gitignore típico de un proyecto KMP
build/
.gradle/
.idea/
*.xcuserstate
local.properties
```

```yaml
# uso de un GitHub Secret dentro de un workflow — la key nunca aparece
# en el código ni en los logs del workflow (GitHub la enmascara automáticamente)
- name: Build with API key
  env:
    API_KEY: ${{ secrets.API_KEY }}
  run: ./gradlew assembleRelease
```

```kotlin
// BuildKonfig: la key se inyecta en tiempo de compilación desde una
// variable de entorno o gradle.properties local (que SÍ está en .gitignore),
// nunca hardcodeada en el código fuente
buildkonfig {
    packageName = "com.timbax.app"
    defaultConfigs {
        buildConfigField(STRING, "API_KEY", System.getenv("API_KEY") ?: "")
    }
}
```

## 4. Matriz de criterio

| Práctica | Usar cuando | NO usar cuando | Trade-off real |
|---|---|---|---|
| **`.gitignore` bien configurado** | Siempre, desde el primer commit del proyecto | — (no hay escenario donde convenga trackear `build/` o `.idea/`) | Cuesta 2 minutos configurarlo al inicio; el costo de no hacerlo crece con cada commit que ensucia el historial |
| **GitHub Secrets (CI)** | La key la necesita un workflow automatizado (build, deploy) | La key es solo para desarrollo local, sin CI de por medio | Mantiene la key fuera del código y de los logs, pero exige configurarla una vez por repo en Settings → Secrets |
| **Variables de entorno locales / BuildKonfig** | La key la necesita el desarrollador para compilar/correr localmente | Nunca conviene hardcodearla directo en un archivo trackeado, ni siquiera "temporalmente para probar" | Exige que cada dev configure su propia copia local (`local.properties` o similar) antes de poder buildear |

Pregunta típica: *"¿qué harías si accidentalmente commiteaste una API key?"* → Rotar la key inmediatamente (invalidarla y generar una nueva) — borrar el commit del historial no alcanza, porque cualquiera que haya clonado el repo antes de la limpieza ya tiene esa key en su copia local; el borrado del historial (`git filter-repo` o similar) es una limpieza de higiene, pero la rotación de la credencial es lo que realmente resuelve el riesgo.

## 5. Caso trampa

**Situación:** Commiteaste por error una API key en un archivo de configuración. Te das cuenta rápido, así que hacés `git reset --soft HEAD~1`, sacás la key del archivo, y volvés a commitear — "problema resuelto, ni siquiera llegué a pushear".

**Por qué la respuesta obvia es incorrecta:** Acá la trampa es sutil pero real: **si nunca pusheaste ese commit**, tenés razón — el `reset` local alcanza, porque la key nunca salió de tu máquina. Pero si en algún momento **sí hiciste push** (aunque haya sido hace 30 segundos, aunque lo hayas "corregido" enseguida con un force-push después), la key ya viajó al remoto — y en el momento en que un commit llega a un repositorio remoto, tenés que asumir que pudo haber sido visto, indexado, o clonado por algo o alguien (incluidos bots que escanean commits públicos de GitHub en tiempo real buscando patrones de credenciales). El reset local prolijo te da una falsa sensación de "nunca pasó", pero no revierte lo que ya salió de tu control. La regla real no es "¿alcancé a corregirlo rápido?" sino "¿ese commit llegó a existir en algún remoto, aunque sea por un instante?" — si la respuesta es sí, la key se rota, sin excepción.

## 6. Conexión con Timbax

Timbax ya usa BuildKonfig para manejar la configuración de Firebase de forma segura (según lo que documentamos en `03_data`), lo cual es exactamente el patrón correcto acá: las credenciales de Firebase nunca están hardcodeadas en un archivo trackeado por Git, sino inyectadas en tiempo de compilación. El día que sumes CI a Timbax (conectando con `github_actions_cicd.md`), esas mismas credenciales pasarían a vivir en GitHub Secrets para que el workflow pueda buildear una release firmada sin que la key aparezca nunca en el repo público — coherente con que Timbax es un proyecto que también funciona como portfolio visible en GitHub.