# Publicación en Play Console

## 1. Qué es

Este archivo cubre la mecánica concreta de publicar en Google Play que todavía no está en `pipeline_de_promocion.md` (que documenta los tracks y la promoción entre ellos) ni en `cicd_multi_ambiente.md` (que documenta automatizar el upload). Acá entran tres piezas: **Play App Signing** — el sistema obligatorio hoy para apps nuevas, donde Google gestiona la clave final que firma lo que llega a cada dispositivo (`app signing key`), mientras vos gestionás una clave propia (`upload key`) que solo sirve para autenticar tus subidas; **staged rollout** — la posibilidad de publicar una release a producción para un porcentaje chico de usuarios primero, y subirlo gradualmente en vez de a todos de una; y el requisito que más golpea a un desarrollador solo o un equipo chico: desde noviembre de 2023, las cuentas de desarrollador **personales** creadas después de esa fecha necesitan completar un test cerrado con un mínimo de testers activos durante un período continuo antes de poder siquiera solicitar acceso a producción — no es opcional, es un gate real de la plataforma.

## 2. El problema que resuelve

Sin entender esta mecánica, un desarrollador que ya tiene la app funcionando y compilada se encuentra con bloqueos que no tienen que ver con su código: Play Console rechaza la solicitud de acceso a producción sin explicar bien por qué, se pierde una `upload key` sin saber que es recuperable, o se publica una versión con un bug crítico al 100% de los usuarios de una sola vez cuando había una opción de mitigar ese riesgo con un rollout gradual. Este archivo cierra la brecha entre "tengo el `.aab` listo" (que es donde termina `cicd_multi_ambiente.md`) y "la app está de verdad disponible y llegando de forma segura a usuarios reales".

## 3. Ejemplo mínimo comentado

```text
Play App Signing — dos claves, dos responsabilidades distintas:

Upload key (la generás y guardás vos):
  - Firma el .aab ANTES de subirlo a Play Console
  - Le prueba a Google que la subida viene de tu cuenta
  - Si se pierde/compromete: se puede resetear desde Play Console SIN
    romper las actualizaciones existentes de la app (justamente porque
    no es la clave que ven los dispositivos finales)

App signing key (la genera y gestiona Google):
  - Firma el APK final que efectivamente llega a cada dispositivo
  - Es la identidad real y permanente de la app ante Android
  - Nunca la ves ni la manejás vos directamente
```

```ruby
# Fastlane — publicar con rollout gradual en vez de al 100% de una vez
lane :deploy_production_gradual do
  gradle(task: "bundleRelease")
  upload_to_play_store(
    track: "production",
    aab: "app/build/outputs/bundle/release/app-release.aab",
    rollout: "0.10" # 10% de los usuarios reciben esta versión primero
  )
end
```

Subir el porcentaje (`0.10` → `0.50` → `1.0`) es un paso deliberado posterior, típicamente disparado a mano una vez confirmado que no hay regresiones — no es algo que conviene automatizar sin un humano mirando métricas primero.

## 4. Matriz de criterio

**Play App Signing (Google gestiona la clave final) vs. autofirmar todo vos:**
- Usar Play App Signing cuando: siempre — hoy es un requisito para apps nuevas en Play Store, y además protege contra el escenario más grave posible (perder la clave que firma la app para siempre, dejando imposible publicar updates).
- Trade-off: le das a Google la custodia de la clave final, a cambio de que perder tu `upload key` deje de ser una catástrofe irrecuperable y pase a ser un trámite de reseteo dentro de Play Console.

**Staged rollout vs. publicar al 100% directo:**
- Usar staged rollout cuando: es cualquier release a producción con cambios de código reales — empezar en un porcentaje chico (1-10%) permite detectar un crash o una regresión grave afectando a una fracción mínima de usuarios, en vez de a todos a la vez.
- Publicar al 100% directo cuando: es una release de contenido/configuración sin riesgo real de romper nada (un cambio de texto, un ajuste de `BuildKonfig` ya validado en testing) — el costo de gestionar el rollout gradual no se justifica si el riesgo real es bajo.
- Trade-off: el rollout gradual agrega pasos manuales y tiempo hasta que el 100% de los usuarios tiene la última versión, a cambio de contener el radio de impacto de cualquier bug que se haya escapado del testing.

**Cuenta de desarrollador personal vs. de organización, respecto al gate de testers:**
- Si la cuenta es personal y fue creada después del 13 de noviembre de 2023: el gate de closed testing con un mínimo de testers activos durante 14 días continuos **es obligatorio** antes de poder solicitar acceso a producción — sin excepción, y sin que el testing interno (`internal testing`) cuente para ese requisito.
- Si la cuenta es de organización (con verificación D-U-N-S): está exenta de este gate específico — puede ir directo del testing interno a producción sin el paso obligatorio de closed testing con mínimo de testers.
- Trade-off: una cuenta de organización requiere el trámite de verificación D-U-N-S, más pesado de tramitar que una cuenta personal — el gate de testers es, en la práctica, el costo de la simplicidad de arrancar con una cuenta personal.

## 5. Caso trampa

Asumir que se puede pasar de `internal testing` directo a `production` porque el botón "Promote release" existe y técnicamente lo permite (tal como se describe el mecanismo en `pipeline_de_promocion.md`):

Para una cuenta de desarrollador **personal** creada después del 13 de noviembre de 2023 — el caso típico de alguien publicando su primera app de forma independiente — Google Play exige haber corrido un test **cerrado** (no alcanza con `internal`) con un mínimo de testers que hayan estado activamente optados-in de forma **continua** durante 14 días, antes de que la opción de solicitar acceso a producción esté siquiera disponible en el Dashboard de Play Console. Ni siquiera importa cuántos testers pasaron por `internal testing`: ese track no cuenta para este requisito en absoluto. Si en algún momento de esos 14 días la cantidad de testers activos cae por debajo del mínimo, la continuidad se rompe y el conteo puede reiniciarse. Y es un requisito por **app**, no por cuenta: los mismos testers que ya validaron una app anterior no "heredan" el progreso en una app nueva — hay que volver a correr el ciclo completo de 14 días para cada app distinta que se publique.

La señal de alarma: planificar el lanzamiento de una app nueva asumiendo que "subo a internal, pruebo un par de días, y promuevo a producción" sin haber verificado primero si la cuenta está sujeta a este gate — el resultado es enterarse del requisito recién cuando Play Console rechaza la solicitud, perdiendo semanas de calendario que se podrían haber usado corriendo el test cerrado en paralelo con el resto del desarrollo.

## 6. Conexión con Timbax

Si la cuenta de desarrollador de Timbax en Google Play es personal y fue creada después de noviembre de 2023, este gate aplica directo y de lleno: antes de poder solicitar producción hace falta reclutar un grupo de testers (compañeros de las partidas de Chinchón/Truco reales son candidatos naturales, ya que además dan feedback genuino sobre el producto) que mantengan la app instalada y activa durante 14 días corridos vía el track `closed testing` — no `internal` — documentado en `pipeline_de_promocion.md`. Es exactamente el tipo de paso que conviene planificar con semanas de anticipación respecto a una fecha de lanzamiento objetivo, y una razón concreta más (además del testing en sí) por la que ese track "closed" del pipeline de promoción no es un paso opcional para saltear rápido, sino frecuentemente un requisito duro de la plataforma. Una vez superado ese gate, el resto (Play App Signing, staged rollout al publicar actualizaciones futuras) aplica igual que para cualquier app ya establecida.