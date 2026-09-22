# DexKot Maven Repository

Repositorio Maven público de las librerías DexKot, servido como sitio estático por
GitHub Pages en **<https://dexkot.github.io/maven>**.

No es un mirror ni un proxy: los artefactos viven versionados en este repo, bajo
`dev/dexkot/mobile/`. Gradle los descarga por HTTPS como de cualquier otro repositorio Maven.

Índice navegable con todos los artefactos y sus versiones: <https://dexkot.github.io/maven>

---

## Agregar el repositorio

### Gradle (Kotlin DSL) — recomendado

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://dexkot.github.io/maven") }
    }
}
```

`google()` y `mavenCentral()` siguen siendo necesarios: acá solo viven los artefactos
`dev.dexkot.mobile`, y sus dependencias transitivas (Kotlin, coroutines, Compose, SQLDelight,
multiplatform-settings, Koin) se resuelven desde los repos públicos.

### Gradle (Groovy DSL)

```groovy
// settings.gradle
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://dexkot.github.io/maven' }
    }
}
```

### Maven (pom.xml)

```xml
<repositories>
  <repository>
    <id>dexkot</id>
    <url>https://dexkot.github.io/maven</url>
  </repository>
</repositories>
```

Sirve para los artefactos JVM/Android, pero los módulos KMP se publican con Gradle Module
Metadata (`.module`) y sus variantes por target. Maven solo lee el `.pom`, así que en ese caso
hay que declarar el artefacto de target explícito (`core-foundation-android`, por ejemplo).
Para consumo KMP real, usar Gradle.

---

## Declarar dependencias

```kotlin
// build.gradle.kts del módulo
dependencies {
    implementation("dev.dexkot.mobile:core-foundation:0.26.0")
    implementation("dev.dexkot.mobile:core-logger:0.26.0")
    implementation("dev.dexkot.mobile:core-ui:0.26.0")
    implementation("dev.dexkot.mobile:core-ui-di-koin:0.26.0")

    // procesador KSP de AutoLog (@AutoLogUseCase, @AutoLogViewModel, …)
    ksp("dev.dexkot.mobile:core-ksp-processor:0.26.0")
}
```

Con version catalog:

```toml
# gradle/libs.versions.toml
[versions]
dexkot = "0.26.0"

[libraries]
dexkot-core-foundation = { module = "dev.dexkot.mobile:core-foundation", version.ref = "dexkot" }
dexkot-core-logger     = { module = "dev.dexkot.mobile:core-logger",     version.ref = "dexkot" }
dexkot-core-ui         = { module = "dev.dexkot.mobile:core-ui",         version.ref = "dexkot" }
dexkot-ksp-processor   = { module = "dev.dexkot.mobile:core-ksp-processor", version.ref = "dexkot" }
```

### KMP: siempre el artefacto padre

Los módulos multiplataforma publican el artefacto padre más una variante por target
(`-android`, `-iosarm64`, `-iossimulatorarm64`). **Declarar siempre el padre** —
`dev.dexkot.mobile:core-foundation` — y dejar que Gradle resuelva el target vía Module
Metadata. Declarar `core-foundation-android` a mano funciona, pero rompe la resolución en
`commonMain` de un proyecto KMP.

```kotlin
// build.gradle.kts de un módulo KMP
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("dev.dexkot.mobile:core-foundation:0.26.0")
            implementation("dev.dexkot.mobile:core-database:0.26.0")
        }
    }
}
```

---

## Qué hay publicado

Group único: **`dev.dexkot.mobile`** · 21 módulos (66 artefactos contando variantes de target).
Todos provienen del monorepo [DexKot/mobile-core](https://github.com/DexKot/mobile-core), que
documenta qué hace cada uno.

| Familia | Módulos |
|---------|---------|
| Base | `core-foundation`, `core-annotation`, `core-ksp-processor` |
| Logging | `core-logger`, `core-logger-firebase`, `core-logger-posthog`, `core-logger-di-koin` |
| Persistencia | `core-database`, `core-database-sqldelight`, `core-preferences` |
| Remote config | `core-remoteconfig`, `core-remoteconfig-firebase`, `core-remoteconfig-posthog`, `core-remoteconfig-di-koin` |
| Background work | `core-work`, `core-work-di-koin` |
| Permisos | `core-permission`, `core-permission-di-koin` |
| UI / navegación | `core-ui`, `core-ui-di-koin` |

Descontinuado: `core-di-koin` quedó en `0.25.0` y se eliminó en `0.26.0`; su `workModule()` se
reemplaza por `workManagerWorkModule` de `core-work-di-koin`.

La fuente de verdad de la lista y de las versiones disponibles es
[`artifacts.json`](./artifacts.json), que también alimenta el índice HTML.

Licencia de los artefactos: MIT (declarada en cada `.pom`).

---

## Estructura del repo

```
.
├── dev/dexkot/mobile/               layout Maven estándar
│   └── <artifactId>/
│       ├── maven-metadata.xml       + .md5 / .sha1 / .sha256 / .sha512
│       └── <version>/
│           ├── <artifactId>-<version>.aar | .jar
│           ├── <artifactId>-<version>.pom
│           ├── <artifactId>-<version>.module      Gradle Module Metadata (KMP)
│           ├── <artifactId>-<version>-sources.jar
│           └── *.md5 / *.sha1 / *.sha256 / *.sha512
├── artifacts.json                   índice de artefactos, versiones y tipo
├── index.html                       página del repositorio (consume artifacts.json)
└── .nojekyll                        evita el pipeline Jekyll de Pages
```

`.nojekyll` es necesario: sin él, GitHub Pages ignora archivos y directorios que empiezan con
`_` y puede alterar la respuesta de los binarios.

---

## Cómo se publica

**Este repo no se edita a mano.** Los artefactos los sube CI desde el repo fuente:

1. En [`mobile-core`](https://github.com/DexKot/mobile-core) se taguea `vX.Y.Z` (semver; el
   workflow valida el formato del tag).
2. El workflow `release` corre build + tests y delega en el workflow reutilizable
   `DexKot/.github/.github/workflows/publish-maven.yml`, autenticado como GitHub App
   (`APP_ID` / `APP_PRIVATE_KEY`).
3. Ese workflow publica los artefactos acá, regenera `maven-metadata.xml` y `artifacts.json`,
   y commitea con el mensaje `publish: mobile-core X.Y.Z`.
4. GitHub Pages redeploya el sitio; la versión nueva queda resoluble en minutos.

Las versiones publicadas son **inmutables**: no se sobreescribe ni se borra una versión ya
publicada. Si una release sale mal, se publica la siguiente (`X.Y.Z+1`).

---

## Troubleshooting

**`Could not find dev.dexkot.mobile:core-xxx:0.26.0`**
- Verificá que el repo esté declarado en `settings.gradle.kts`. Si el proyecto usa
  `repositoriesMode = FAIL_ON_PROJECT_REPOS` (el default de muchos templates), declararlo en el
  `build.gradle.kts` del módulo no alcanza.
- Confirmá que la versión exista en [`artifacts.json`](./artifacts.json) — no todos los módulos
  comparten historial de versiones (los KMP con targets iOS arrancan en `0.24.1`).
- Si acabás de taguear, esperá a que termine el deploy de Pages.

**Gradle sigue viendo la versión vieja**
Pages cachea en CDN y Gradle cachea localmente. Forzá con:

```bash
./gradlew build --refresh-dependencies
```

**`No matching variant` en un proyecto KMP**
Estás declarando una variante de target (`-android`, `-iosarm64`) en `commonMain`. Usá el
artefacto padre.

**404 en el navegador al abrir una ruta de `dev/…`**
El listado de directorios no está habilitado en Pages. Navegá desde
[el índice](https://dexkot.github.io/maven) o pedí el archivo por su ruta completa.

---

## Enlaces

- Índice de artefactos: <https://dexkot.github.io/maven>
- Código fuente de las librerías: <https://github.com/DexKot/mobile-core>
- Documentación y skills: <https://github.com/DexKot/docs>
