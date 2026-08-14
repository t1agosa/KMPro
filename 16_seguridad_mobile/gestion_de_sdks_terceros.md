# Gestión de SDKs de terceros

## 1. Qué es

Es la disciplina de tratar cada SDK/librería externa que agregás a tu app (Firebase, Ktor, un SDK de ads, un SDK de analytics, una librería de pagos) como una **superficie de riesgo**, no solo como una dependencia técnica más. Cada SDK de terceros que entra a tu binario trae su propio código, sus propios permisos, sus propias conexiones de red, y potencialmente sus propias vulnerabilidades — y todo eso corre con los mismos privilegios que el resto de tu app.

Incluye: auditar qué datos recolecta cada SDK y hacia dónde los manda, mantener las versiones actualizadas (parches de seguridad), minimizar la cantidad de SDKs a los estrictamente necesarios, y revisar permisos que cada SDK agrega al manifest/Info.plist.

## 2. El problema que resuelve

Cuando agregás un SDK de terceros, no solo importás su funcionalidad — importás también su código, sus dependencias transitivas, y su comportamiento de red, sin control directo sobre ninguno de los tres. Esto se llama **riesgo de supply chain** (cadena de suministro): un SDK comprometido (por un ataque al propio proveedor, o por una versión maliciosa publicada por error o intencionalmente) se ejecuta con los mismos permisos que tu código — puede leer el mismo almacenamiento, hacer las mismas llamadas de red, acceder a la misma cámara/ubicación si tu app tiene esos permisos.

El problema práctico más común no es un ataque sofisticado, sino algo más simple: un SDK de analytics o ads que recolecta más datos de los que tu política de privacidad declara, exponiendo a la app a problemas de compliance (GDPR, políticas de Google Play/App Store) sin que el equipo lo haya decidido conscientemente — simplemente vino por default en el SDK.

## 3. Ejemplo mínimo comentado

No hay "código" central acá — es proceso y configuración. Un ejemplo concreto de una práctica aplicable: revisar qué permisos agrega un SDK al manifest de Android, y confirmar que sean los esperados.

```kotlin
// build.gradle.kts (androidApp) — versión pineada explícita, no rango abierto
dependencies {
    implementation("com.google.firebase:firebase-auth-ktx:23.1.0") // versión fija
    // implementation("com.google.firebase:firebase-auth-ktx:23.+") // evitar: rango abierto
}
```

```xml
<!-- AndroidManifest.xml generado — revisar qué agregó cada SDK -->
<!-- ./gradlew :app:processReleaseManifest y mirar el merged manifest -->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<!-- ¿este permiso lo pediste vos, o lo trajo un SDK de ads/analytics? -->
```

```kotlin
// commonMain — Koin: instanciar el cliente del SDK en un solo punto,
// para poder auditar/reemplazar fácilmente sin tocar el resto del código
val thirdPartySdkModule = module {
    single { AnalyticsClient.initialize(apiKey = BuildKonfig.ANALYTICS_KEY) }
}
```

## 4. Matriz de criterio

**Auditar/limitar SDKs de terceros cuando:**
- El SDK pide permisos que no son evidentemente necesarios para su función declarada (un SDK de analytics pidiendo acceso a contactos o ubicación fina).
- El SDK va a tener acceso a datos sensibles del usuario — cualquier SDK que toque autenticación, pagos, o datos de salud/financieros necesita más escrutinio que uno puramente visual (un SDK de animaciones, por ejemplo).
- El proyecto está en un dominio regulado (fintech, salud) donde cada dependencia externa entra al alcance de una auditoría de compliance.

**Confiar con revisión estándar (sin proceso extra) cuando:**
- Es una librería ampliamente adoptada, con mantenimiento activo y buena reputación en el ecosistema (Firebase, Ktor, Koin) — el riesgo no desaparece, pero el costo/beneficio de una auditoría profunda no se justifica para cada dependencia de este calibre.
- El SDK no toca red ni almacenamiento persistente (ej: una librería puramente de UI/utilidades sin I/O).

**Pinear versiones exactas (no rangos abiertos) siempre:** usar `23.1.0` en vez de `23.+` — un rango abierto significa que un `./gradlew build` en una fecha distinta puede traer una versión distinta sin que nadie lo decidiera explícitamente, rompiendo reproducibilidad de builds y abriendo la puerta a que una versión comprometida entre sin aviso.

**Trade-off real:** cada SDK que sumás reduce tiempo de desarrollo a corto plazo, pero aumenta superficie de ataque, tamaño del binario, y carga de mantenimiento a largo plazo (cada uno necesita actualizarse, cada uno puede tener breaking changes). La pregunta correcta no es "¿este SDK es seguro?" sino "¿el valor que aporta justifica la superficie de riesgo que agrega, versus construirlo nosotros o no tenerlo?".

## 5. Caso trampa

"Actualicé todos los SDKs a la última versión, así que estoy protegido" — actualizar ciegamente sin revisar el changelog es tan riesgoso como no actualizar: una nueva versión puede agregar telemetría nueva, cambiar permisos requeridos, o (en casos raros pero reales) haber sido comprometida en un ataque de supply chain donde el paquete legítimo fue reemplazado por una versión maliciosa en el propio repositorio (Maven Central, CocoaPods). La práctica correcta no es "actualizar siempre lo último" ni "nunca actualizar" — es actualizar deliberadamente, revisando el changelog/release notes, y idealmente con una ventana de espera antes de adoptar una versión recién publicada (para que la comunidad tenga tiempo de detectar problemas).

Otro caso trampa: pensar que "es un SDK de una empresa grande y conocida, así que no hace falta revisar permisos". El tamaño de la empresa no elimina el riesgo — SDKs de proveedores grandes de ads/analytics son justamente los que más datos recolectan por diseño (es su modelo de negocio), no por descuido. La confianza en la reputación del proveedor no reemplaza la revisión de qué datos concretos se están enviando.

## 6. Conexión con arquitectura real (Timbax)

En Timbax, los SDKs de terceros están concentrados en capas bien definidas gracias a Clean Architecture: Firebase (vía GitLive SDK) vive en `data` detrás de las interfaces de `domain` — si mañana hubiera que auditar o reemplazar Firebase Auth por otro proveedor, el cambio queda contenido a `RepositoryImpl` y al módulo de Koin correspondiente, sin tocar `domain` ni `presentation`. Esto es un beneficio de seguridad indirecto de la arquitectura ya documentada en `04_di/koin_fundamentos_y_scopes.md`: menos superficie de un SDK filtrada a través de toda la codebase significa menos lugares que auditar cuando cambia una versión o se descubre un problema. Timbax hoy mantiene su lista de SDKs deliberadamente chica (Firebase, Ktor, SQLDelight, Koin) — coincide con el criterio de la sección 4 de limitar dependencias a lo estrictamente necesario, sin sumar SDKs de ads o analytics de terceros que no aportan al core del producto.