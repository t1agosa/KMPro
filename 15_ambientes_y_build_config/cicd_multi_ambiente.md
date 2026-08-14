# CI/CD Multi-Ambiente

## 1. Qué es

Es extender el CI básico que ya vimos en `github_actions_cicd.md` (compilar, testear, lint en cada PR) para que, además, **automatice el deploy** hacia los distintos ambientes/tracks (internal, closed, TestFlight, producción) según qué rama o evento disparó el workflow — cerrando el círculo entre `flavors_y_schemes_por_plataforma.md`, `buildkonfig_multi_ambiente.md` y `pipeline_de_promocion.md`.

La herramienta estándar de la industria para orquestar el build+firma+upload en ambas plataformas es **Fastlane** (Ruby), invocada desde GitHub Actions. Fastlane abstrae comandos específicos de cada store (`upload_to_play_store`, `upload_to_testflight`) detrás de "lanes" reutilizables.

## 2. El problema que resuelve

Sin esto, cada release depende de que una persona, a mano, en su máquina: compile con la configuración correcta, firme con el keystore/certificado correcto, y suba manualmente a Play Console/App Store Connect — un proceso lento, propenso a error humano (subir el flavor equivocado, olvidar incrementar el `versionCode`), y que no queda documentado como código versionado.

## 3. Ejemplo mínimo comentado

**Workflow que dispara distinto ambiente según la rama** (`.github/workflows/release.yml`):

```yaml
name: Release

on:
  push:
    branches: [develop, main]

jobs:
  deploy-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      # develop -> sube a internal testing; main -> sube a closed testing
      - name: Build y deploy Android
        env:
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          PLAY_SERVICE_ACCOUNT_JSON: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
        run: |
          if [ "${{ github.ref_name }}" = "develop" ]; then
            bundle exec fastlane android deploy_internal
          else
            bundle exec fastlane android deploy_closed
          fi
```

**Fastlane lane (`android/fastlane/Fastfile`)** que sube el `.aab` a un track específico:

```ruby
platform :android do
  desc "Deploy a internal testing"
  lane :deploy_internal do
    gradle(task: "bundleRelease")
    upload_to_play_store(
      track: "internal", # también puede ser: closed, open, production
      aab: "app/build/outputs/bundle/release/app-release.aab",
      release_status: "completed"
    )
  end
end
```

Para iOS, el equivalente sube a TestFlight vía `upload_to_testflight`, usando una API Key de App Store Connect (rol *App Manager*) almacenada como secret, no un usuario/contraseña.

## 4. Matriz de criterio

| Escenario | Automatizar en CI/CD | NO automatizar / manual |
|---|---|---|
| Subir a `internal testing` / TestFlight internal en cada merge a una rama de integración | Sí — es de bajo riesgo (audiencia acotada, sin revisión de la store) y da feedback rápido | — |
| Promover de `closed` a `production` (Android) o agregar testers externos (iOS) | Depende del apetito de riesgo del equipo: puede automatizarse con un trigger manual (`workflow_dispatch`) para mantener un humano en el loop antes de impactar producción | Manual desde la consola, como paso deliberado post-validación |
| Manejo de secretos (keystore, certificados, API keys de las stores) | Siempre vía GitHub Secrets inyectados en runtime — nunca committeados (ver `secretos_gitignore.md`) | — |
| Proyecto chico/personal en etapa muy temprana | El CI básico (test+lint) puede alcanzar por un tiempo; el deploy automatizado se justifica cuando el ritmo de releases lo amerita | Deploy manual ocasional |

## 5. Caso trampa

**"El workflow de deploy a Android funciona perfecto, pero el `versionCode` quedó duplicado y Play Console rechaza el upload."**

La trampa: automatizar el deploy sin automatizar también el **incremento del `versionCode`/build number** es un error común — si cada corrida de CI usa el mismo valor hardcodeado (o el `versionCode` que quedó en el repo de la corrida anterior), la segunda subida falla porque Play Console exige que cada `.aab` tenga un `versionCode` estrictamente mayor al anterior. La solución típica es derivar el `versionCode` de algo que cambie solo (el número de run de GitHub Actions vía `${{ github.run_number }}`, o un contador gestionado por Fastlane), nunca de un valor fijo en el `build.gradle.kts` que alguien tiene que acordarse de subir a mano.

## 6. Conexión con arquitectura real (Timbax)

Para Timbax, un pipeline realista encadenaría los tres archivos anteriores del package: el CI compila usando el flavor/scheme correspondiente (`flavors_y_schemes_por_plataforma.md`), que a su vez determina qué valores de BuildKonfig se usan (`buildkonfig_multi_ambiente.md`), y el resultado de ese build es lo que efectivamente avanza por el pipeline de promoción (`pipeline_de_promocion.md`) — todo disparado automáticamente, sin pasos manuales salvo la decisión consciente de promover a producción.