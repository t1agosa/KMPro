# Flavors y Schemes por Plataforma

## 1. Qué es

Son los mecanismos nativos de cada plataforma para manejar **múltiples variantes de la misma app** (dev/staging/prod, free/premium, etc.) sin duplicar el proyecto entero.

- **Android → Product Flavors** (Gradle): se declaran en el `build.gradle.kts` del módulo `application`, agrupados en `flavorDimensions`. Cada flavor puede tener su propio `applicationIdSuffix`, recursos, y hasta código fuente propio (`src/dev/kotlin/...`).
- **iOS → Xcode Schemes**: un scheme define qué target, qué configuración de build (Debug/Release) y qué variables de entorno se usan al compilar/correr. No son lo mismo que los *Build Configurations* de Xcode, aunque suelen ir de la mano (un scheme "Staging" típicamente apunta a una configuración "Staging").

**Importante en KMP:** estos dos mecanismos son completamente independientes entre sí — Gradle no sabe qué scheme eligió Xcode, y Xcode no sabe qué flavor existe en Gradle. No hay sincronización automática.

## 2. El problema que resuelve

Sin flavors/schemes, para tener una versión "dev" (apuntando a un backend de pruebas) y una "prod" (apuntando al backend real) tendrías que:
- Comentar/descomentar URLs a mano antes de cada build, con el riesgo real de subir un build a producción apuntando al backend de staging.
- O mantener dos proyectos separados, duplicando todo el mantenimiento.

Flavors y schemes permiten declarar esas variantes una sola vez, y elegir cuál compilar desde un selector (Android Studio / Xcode), sin tocar código a mano cada vez.

## 3. Ejemplo mínimo comentado

**Android — declarando flavors (módulo `composeApp` o `androidApp`):**

```kotlin
android {
    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            resValue("string", "app_name", "Timbax Dev")
        }
        create("prod") {
            dimension = "environment"
            resValue("string", "app_name", "Timbax")
        }
    }
}
```

**iOS — pasando el flavor elegido en Xcode hacia Gradle**, vía una variable de entorno definida en el scheme (User-Defined Setting), que el script de build de Xcode reenvía al comando de Gradle que compila el framework KMP:

```bash
# Build Phase "Compile Kotlin Framework" del proyecto Xcode
./gradlew :composeApp:embedAndSignAppleFrameworkForXcode -PflavorEnvironment=$ENVIRONMENT
```

Acá `$ENVIRONMENT` viene de un User-Defined Setting del scheme ("dev" o "prod"), y `-PflavorEnvironment` es una property que tu `build.gradle.kts` compartido puede leer para decidir, por ejemplo, qué valor de BuildKonfig usar.

## 4. Matriz de criterio

| Escenario | Usar flavors/schemes | NO usar / alternativa |
|---|---|---|
| Necesitás distinto backend/API key por ambiente (dev/staging/prod) | Sí, es el caso de uso central | — |
| Necesitás distinto ícono/nombre de app por ambiente para poder tener dev y prod instaladas a la vez | Sí (flavor con `applicationIdSuffix` + `resValue`; en iOS, scheme con Bundle ID distinto) | — |
| Necesitás una variante "free/premium" con código de negocio realmente distinto (no solo config) | Evaluar: si la diferencia es de lógica de negocio compleja, considerá separar en módulos Gradle en vez de flavors — los flavors escalan mal cuando además de config hay ramas de código enteras | Módulos separados con interfaces compartidas |
| Módulo KMP puro (librería, sin target `application`) usando el nuevo plugin `com.android.kotlin.multiplatform.library` | No aplica — ese plugin usa arquitectura single-variant y **no soporta** product flavors ni build types | Manejar la variante vía property de Gradle inyectada desde afuera, no vía flavor |
| Diferencia es solo un valor (URL, flag), no estructura de código | BuildKonfig alcanza, no hace falta flavor de Gradle completo | Ver `buildkonfig_multi_ambiente.md` |

## 5. Caso trampa

**"Le agregué product flavors a mi módulo compartido de KMP y no me compila."**

La trampa: product flavors son una feature del plugin `com.android.application` (o del clásico `com.android.library`), pensada para el mundo Android tradicional. Si tu módulo `shared`/`composeApp` usa el nuevo plugin dedicado de Android-KMP (`com.android.kotlin.multiplatform.library`, pensado específicamente para librerías KMP puras), ese plugin usa una **arquitectura single-variant** y **no soporta build types ni product flavors** — es una limitación intencional para simplificar el build, no un bug tuyo.

La solución típica en ese caso no es "forzar" flavors donde no van, sino inyectar la variante como property de Gradle (`-PflavorEnvironment=dev`) leída en tiempo de configuración, y dejar que sea esa property la que decida qué valores usa BuildKonfig — el flavor "vive" en la property, no en un mecanismo de Android Gradle Plugin.

## 6. Conexión con arquitectura real (Timbax)

Si Timbax necesitara distinguir entre un backend de pruebas (para probar features de Firebase sin ensuciar datos reales de usuarios) y el backend de producción, el camino sería:
- **Android**: flavors `dev`/`prod` en el módulo de la app (no en el módulo `shared` si ese usa el plugin KMP puro), cada uno apuntando a un proyecto Firebase distinto (vía `google-services.json` por flavor).
- **iOS**: dos schemes, cada uno con su propio Bundle ID y su `GoogleService-Info.plist`.
- **Capa compartida (`domain`/`data`)**: no necesita saber nada de esto — sigue recibiendo la `baseUrl` o config ya resuelta vía BuildKonfig, manteniendo la Dependency Rule intacta (domain no sabe si está corriendo en "dev" o "prod", solo usa la configuración que le inyectan).