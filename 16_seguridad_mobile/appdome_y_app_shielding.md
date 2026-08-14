# Appdome y app shielding (RASP)

## 1. Qué es

**RASP (Runtime Application Self-Protection)** es la categoría de seguridad que protege una app *mientras corre*, detectando y reaccionando a ataques en tiempo real — a diferencia de defensas "estáticas" (revisar el código antes de publicar). **Appdome** es una plataforma **no-code** que se engancha al pipeline de CI/CD y le inyecta estas protecciones al binario ya compilado, sin que el equipo de desarrollo escriba código de seguridad ni integre un SDK propio: Appdome es una plataforma de seguridad mobile sin código que protege apps Android y iOS sin requerir SDKs, cambios de código, ni recursos de desarrollo especializados — usa tecnología patentada llamada Fusion para inyectar capas de seguridad directamente en los binarios compilados de la app a través de pipelines de CI/CD.

Su producto RASP se llama **ONEShield**: ONEShield es la solución de runtime application self-protection de Appdome, que defiende contra reversing dinámico, tampering, debugging, uso de emuladores, hooking frameworks como Frida y Xposed, y ataques de repackaging — opera sin integración de código ni SDK.

## 2. El problema que resuelve

Play Integrity (ver archivo anterior) te da un **veredicto puntual**: "en este momento, este dispositivo/binario parece genuino". Pero no hace nada activo si un atacante engancha un debugger en tu app corriendo, o usa Frida para hookear funciones en runtime y modificar el comportamiento de la app *mientras el usuario la usa*. RASP cubre esa brecha: permite agregar features de runtime application self-protection como anti-tampering, anti-reversing, anti-debugging, ofuscación de código, cifrado de datos, prevención de jailbreak/root, prevención de MitM, y más, para prevenir amenazas maliciosas a usuarios, apps y datos móviles.

El problema de fondo que resuelve Appdome específicamente (más allá de RASP en general) es de **costo de ingeniería**: implementar estas protecciones a mano (ofuscación, detección de debugger, detección de hooking frameworks) es trabajo especializado y constante — cada nueva técnica de ataque requiere actualizar la defensa. Appdome usa IA para construir cualquiera de sus más de 400 features de RASP y app shielding con conciencia de amenazas, incluyendo anti-tampering, anti-debugging, anti-repackaging, anti-resigning, detección de jailbreak/root y más, dentro del pipeline de CI/CD, para que el equipo de desarrollo mobile no tenga que hacerlo.

## 3. Ejemplo mínimo comentado

Esto **no es código que escribas en Kotlin** — es configuración de pipeline. El punto central del "no-code" es que no hay snippet de Kotlin/Swift que mostrar: se sube el `.apk`/`.ipa` ya compilado a la plataforma de Appdome (o se integra en el paso de CI/CD), se elige qué protecciones aplicar desde una consola, y Appdome devuelve el binario ya blindado.

```yaml
# .github/workflows/release.yml — paso conceptual, no oficial
# se integra DESPUÉS del build normal, ANTES de publicar a las stores
- name: Build release APK
  run: ./gradlew assembleRelease

- name: Shield with Appdome
  uses: appdome/appdome-github-action@v1  # acción ilustrativa
  with:
    app_path: app/build/outputs/apk/release/app-release.apk
    fusion_set: ${{ secrets.APPDOME_FUSION_SET_ID }}
    api_key: ${{ secrets.APPDOME_API_KEY }}
  # output: app-release-shielded.apk, listo para firmar y publicar
```

La app que se construye para Appdome puede armarse con cualquier herramienta nativa, como Xcode para iOS o Android Studio, u otro framework, incluyendo frameworks híbridos y cross-platform como Maui, Xamarin, Cordova, React Native, y Flutter. Compose Multiplatform genera un binario nativo estándar (APK/AAB para Android, framework para iOS embebido en un proyecto Xcode), así que encaja en esa misma categoría de "cualquier binario nativo" sin requerir soporte explícito de KMP.

## 4. Matriz de criterio

**Usar Appdome (o herramientas RASP equivalentes) cuando:**
- La app maneja datos regulados o de alto valor donde el sector espera este nivel de protección por default — banca, fintech, salud. Promon SHIELD es un vendor de app shielding usado fuertemente en fintech y banca móvil.
- El equipo no tiene capacidad ni tiempo de mantener ofuscación/anti-tampering a mano, y el presupuesto permite una herramienta enterprise (Appdome usa pricing enterprise custom basado en la cantidad de apps y defensas requeridas — no hay tier gratuito ni plan self-serve, hay que contactar a ventas).
- Se necesita evidencia de compliance ante auditorías (certificación de que cada build cumple ciertas políticas anti-fraude/ciber).

**NO usar (o evaluar alternativas más livianas) cuando:**
- Es una app de consumo sin datos sensibles ni economía en juego — Timbax es el ejemplo típico: el "peor caso" de un binario tampereado no representa un riesgo de negocio real.
- El presupuesto no alcanza para una herramienta enterprise sin tier gratuito. Existen alternativas con footprint más chico y opciones gratuitas parciales, como Talsec RASP+, que ofrece mobile RASP y detección de amenazas en runtime con un tier gratuito (freeRASP) y un SDK pago (AppiCrypt) — footprint más simple que Appdome, enfocado en RASP en vez del stack completo de fraude/bots.
- El equipo prefiere control fino a nivel de código sobre "caja negra" de terceros — la alternativa de ofuscación a nivel compilador (Guardsquare con DexGuard + iXGuard, ofuscación y hardening a nivel de compilador para Android e iOS — más fuerte en protección profunda de código y ofuscación de bytecode, pero requiere integración en etapa de build en vez de no-code) da más control pero exige integración manual en el pipeline de build.

**Trade-off real:** no-code significa velocidad de adopción y cero mantenimiento de la lógica de seguridad en sí, pero también significa dependencia de un tercero para toda la superficie de seguridad runtime, con pricing enterprise y sin transparencia total de cómo se instrumenta el binario. Es lo opuesto en filosofía a "dueño de tu propio código" — para un equipo chico, el costo/beneficio rara vez cierra salvo que el dominio (fintech, salud) lo exija.

## 5. Caso trampa

"Ya usamos Play Integrity, no necesitamos RASP" — son capas distintas que resuelven momentos distintos del ataque. Play Integrity te dice **si podés confiar en este dispositivo/binario en el momento del request**; RASP protege la app **mientras corre**, incluso si el dispositivo pasó el chequeo de integridad inicial. Un atacante podría pasar la verificación de integridad al arrancar la app y luego, ya con la app corriendo, engancharle un debugger o un framework de hooking para interceptar lógica de negocio en memoria — eso Play Integrity no lo detecta porque ya emitió su veredicto antes de que empezara el ataque. Son complementarios, no sustitutos: Play Integrity es un chequeo puntual server-verificado; RASP es defensa continua en el dispositivo mismo.

Otro caso trampa: pensar que "no-code" significa "sin riesgo de romper la app". Blindar un binario después del build agrega una capa que puede interactuar mal con el propio código de la app (falsos positivos de anti-debugging en dispositivos legítimos, incompatibilidades con ciertas arquitecturas — ONEShield de Appdome solo soporta arquitecturas ARM de 64 bits). Hay que testear el binario shielded, no asumir que es un paso transparente sin efectos secundarios.

## 6. Conexión con arquitectura real (Timbax)

Timbax no usa Appdome ni ninguna herramienta de RASP hoy, y coincide directamente con el criterio de "NO usar" de la sección 4: es una app de consumo (tracker de puntajes de juegos de cartas) sin datos financieros ni PII regulada, sin economía competitiva server-validada — el mismo razonamiento que llevó a no implementar Play Integrity (`play_integrity_api.md`) ni certificate pinning explícito (`cifrado_datos_en_transito.md`). Los tres archivos de este package hasta acá forman un patrón consistente de criterio: cada capa de seguridad extra (pinning, integrity attestation, RASP) tiene un costo operativo real, y para el perfil de riesgo de Timbax ese costo no se justifica — la protección de base (TLS estándar, Firebase Auth, reglas de seguridad de Firestore/Firebase) es proporcional al dominio de la app.