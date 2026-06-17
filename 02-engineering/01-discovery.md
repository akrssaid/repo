# Engineering Discovery Method

This is the canonical method for reading an unfamiliar Android Gradle project from zero context to
a complete factual picture of its build system, modules, and runtime entry points. It is the
technique behind `01-core/prompts/01-discovery.md` (phase P1) and is re-run partially whenever the
build files change materially. Follow the reading order exactly — it is sequenced so that each file
gives you the vocabulary needed to interpret the next. Record everything you extract; never rely on
memory of "what the build probably does".

## P1 Build Policy (canonical in `01-core/CONVENTIONS.md`)

Discovery is read-mostly, not execute-nothing. In P1 you **are permitted** to run read-only Gradle
commands — `./gradlew --version`, `./gradlew projects`, `./gradlew :app:dependencies` — plus **one
optional `assembleDebug` attempt**. If that build attempt fails, document the failure verbatim in
`docs/factory/audits/DISCOVERY.md` as a pre-finding and move on — P1 never fixes builds. Prompt
`01-core/prompts/01-discovery.md` applies the same rule; the P1 gate is "builds OR build failure
documented", not "builds".

## Reading Order (mandatory sequence)

| Step | File | Why this position |
|---|---|---|
| 1 | `settings.gradle` or `settings.gradle.kts` | Defines module set, repositories, plugin management — the project's table of contents |
| 2 | `gradle/libs.versions.toml` | Version catalog (if present) — decodes every alias used in build files |
| 3 | Root `build.gradle(.kts)` | Plugin versions, project-wide config, buildscript-era classpath deps |
| 4 | Each module `build.gradle(.kts)`, `:app` first | Per-module SDK levels, features, flavors, dependencies |
| 5 | `gradle/wrapper/gradle-wrapper.properties` | Gradle version — caps everything else |
| 6 | Every `src/*/AndroidManifest.xml` (main first, then flavor/buildType variants) | Components, permissions, app entry |
| 7 | The `Application` class (from manifest `android:name`) | SDK initialization, DI bootstrap, global state |

### Step 1 — settings file

Extract, verbatim:

- **Module list**: every `include(":...")` line. Note nesting (`:feature:home` implies a directory
  `feature/home/`).
- **`rootProject.name`** — use this as the project identifier in `docs/factory/PROJECT_STATE.md`.
- **`pluginManagement { repositories { ... } }`** and **`dependencyResolutionManagement`**:
  list repositories in order. `jcenter()` here is an immediate red flag (see table below).
  `RepositoriesMode.FAIL_ON_PROJECT_REPOS` present = modern hygiene; absent = check each module
  for its own `repositories` block.
- Any `includeBuild(...)` (composite builds) — these are separate Gradle builds; discover each
  recursively.

### Step 2 — version catalog

If `gradle/libs.versions.toml` exists, extract the `[versions]` table wholesale — it is the single
fastest way to learn AGP, Kotlin, Compose, and library versions. If it does **not** exist, note
"no version catalog" as a modernization candidate (feeds
`02-engineering/03-dependency-analysis.md`, version-catalog adoption assessment) and expect
versions to be scattered: root `ext {}` blocks, `buildSrc/` Kotlin constants, or hardcoded strings
in each module.

### Step 3 — root build file

Extract:

- **AGP version** — from `plugins { id("com.android.application") version "X" }`, the catalog, or
  legacy `classpath("com.android.tools.build:gradle:X")` in `buildscript {}`. Legacy `buildscript`
  + `allprojects { repositories }` style = pre-catalog era, note as modernization candidate.
- **Kotlin plugin version**, and whether `kotlin-kapt`, `kotlin-android-extensions`, or
  `com.google.devtools.ksp` appear anywhere.
- Project-wide plugins: Hilt, Google Services, Crashlytics, Firebase Perf, protobuf, etc.

### Step 4 — module build files (`:app` first)

For **every** Android module extract this exact field set into a table:

| Field | Where in the build file |
|---|---|
| `namespace` | `android { namespace = }` (if absent, package comes from manifest — pre-AGP-8 smell) |
| `compileSdk` / `targetSdk` / `minSdk` | `android {}` / `defaultConfig {}` |
| `versionCode` / `versionName` | `defaultConfig {}` (app module only) |
| `applicationId` (+ flavor suffixes) | `defaultConfig` and each flavor's `applicationIdSuffix` |
| Java/Kotlin level | `compileOptions`, `kotlinOptions.jvmTarget` or `kotlin { jvmToolchain() }` |
| `buildFeatures` | `compose`, `viewBinding`, `dataBinding`, `buildConfig` booleans |
| `buildTypes` | names, `minifyEnabled`, `shrinkResources`, `proguardFiles`, debuggable variants |
| `productFlavors` + `flavorDimensions` | names, per-flavor config diffs (applicationId, fields, deps) |
| `signingConfigs` | **record presence and source only** — e.g., "reads `keystore.properties`" or "env vars on CI". **Never copy passwords/aliases/paths-to-keystores into audit docs.** |
| Native code | `externalNativeBuild { cmake/ndkBuild }`, `ndkVersion`, `sourceSets` with `jniLibs`, any `.so` under `src/main/jniLibs/` |
| `packaging` / `splits` / `bundle` blocks | ABI filters, language splits — release-shape facts |
| Plugins applied | full list per module |

### Step 5 — wrapper

```bash
cat gradle/wrapper/gradle-wrapper.properties
# distributionUrl=...gradle-8.13-bin.zip  → Gradle 8.13
./gradlew --version          # confirms Gradle, JVM, Kotlin (embedded) actually used
```

```powershell
Get-Content gradle\wrapper\gradle-wrapper.properties
# distributionUrl=...gradle-8.13-bin.zip  → Gradle 8.13
.\gradlew.bat --version      # confirms Gradle, JVM, Kotlin (embedded) actually used
```

Cross-check Gradle ↔ AGP ↔ Kotlin ↔ JDK compatibility against
`08-knowledge/android/version-matrix.md`. A mismatch is an immediate Critical ENG- finding because
it blocks every other migration.

### Step 6 — manifests

Read `src/main/AndroidManifest.xml` for every module, plus `src/<flavor>/AndroidManifest.xml` and
`src/debug|release/AndroidManifest.xml` overlays. Extract: `android:name` of the Application
class, launcher activity, permission list, and component counts (full component analysis is method
`02-engineering/05-service-inventory.md`; permission analysis is `06-manifest-audit.md` — at
discovery you only record the raw lists).

### Step 7 — Application class

Open the class named in `<application android:name=...>`. Record every SDK initialized in
`onCreate()`, the DI bootstrap (`@HiltAndroidApp`, `startKoin`, `DaggerAppComponent.create()`), and
any `StrictMode`, multidex, or process-name branching. This is the seed list for
`02-engineering/04-sdk-inventory.md`.

## Detection Heuristics

| Question | How to answer |
|---|---|
| Compose or XML UI? | `buildFeatures { compose = true }` and `androidx.compose.*` deps = Compose. `res/layout/*.xml` count > 0 = XML (both can coexist — record the ratio, see UI-ratio commands below) |
| Multi-module topology | From settings include list + each module's `implementation(project(":x"))` edges; render in `02-engineering/02-architecture-mapping.md` diagram |
| Native code present? | Any `jniLibs` dir, `externalNativeBuild`, or `.so` inside packaged AARs (`./gradlew :app:dependencies` then check known native libs). Native = 16 KB page-size compatibility check required (baseline rule) |
| Flavors used for what? | Read per-flavor `applicationIdSuffix`, `buildConfigField`, manifest overlays — typical axes: free/pro, store (gplay/huawei/rustore), region |
| CI present? | `.github/workflows/`, `.gitlab-ci.yml`, `bitrise.yml`, `Jenkinsfile` at repo root (absence = roadmap item, `02-engineering/12-ci-setup.md`) |

**UI-ratio count** (Compose vs XML coexistence):

```bash
grep -rl --include="*.kt" "@Composable" app/src | wc -l    # files containing composables
find app/src/main/res -path "*/layout*/*.xml" | wc -l       # layout XML files
```

```powershell
(Get-ChildItem -Recurse app\src -Filter *.kt | Select-String -Pattern "@Composable" -List).Count
(Get-ChildItem -Recurse app\src\main\res -Filter *.xml | Where-Object DirectoryName -match "layout").Count
```

## Commands

All Gradle commands below are read-only and P1-permitted (see P1 Build Policy above).

```bash
./gradlew projects                       # authoritative module tree
./gradlew :app:dependencies --configuration releaseRuntimeClasspath > docs/factory/audits/deps-releaseRuntimeClasspath.txt
./gradlew :app:dependencies --configuration debugRuntimeClasspath  > docs/factory/audits/deps-debugRuntimeClasspath.txt   # diff reveals debug-only tooling
./gradlew --version                      # Gradle + JVM in use
./gradlew :app:tasks --group=build       # discover custom build tasks
git log --oneline -15                    # recent activity; last touch date
git log -1 --format=%ci -- app/build.gradle*   # when the build was last modified
```

On Windows PowerShell, substitute `.\gradlew.bat` for `./gradlew`; the `git` commands are
identical. Dump naming is canonical: `docs/factory/audits/deps-<config>.txt` (full procedure in
`02-engineering/03-dependency-analysis.md` Step 1).

If the project does not build at all, still complete steps 1–7 (they are read-only) and record the
build failure verbatim as the first Critical finding — discovery never requires a green build.

## Red Flags Table

| Signal | Where seen | Why it matters | Severity |
|---|---|---|---|
| `jcenter()` | settings/root repositories | JCenter is read-only/sunset; artifacts can vanish; blocks new resolution | High |
| `kotlin-kapt` plugin | module plugins | Kapt is in maintenance; slow; KSP is the baseline. Map each kapt processor to its KSP equivalent | Medium |
| `kotlin-android-extensions` / synthetics | module plugins / `kotlinx.android.synthetic` imports | Removed in Kotlin 1.8+; hard-blocks Kotlin migration | High |
| `compile` / `testCompile` configurations | dependencies block | Removed in AGP 7+; hard-blocks AGP migration | High |
| Dynamic versions (`"1.+"`, `"+"`, `latest.release`) | dependencies | Non-reproducible builds; silent breakage | High |
| `targetSdk` < current Play deadline (35 as of 2026-06; deadline cycles every August) | defaultConfig | App updates get blocked by Google Play | Critical |
| `minifyEnabled false` on release | buildTypes | Shipping unshrunk, unobfuscated release | Medium |
| `android.useAndroidX=false` or jetifier still needed | `gradle.properties` | Support-library era project; large migration ahead | High |
| Checked-in `keystore.jks` / passwords in build files | repo tree / build files | Secret leakage. Flag location; do not print contents | Critical |
| `multiDexEnabled` with minSdk ≥ 21 | defaultConfig | Dead config; harmless but signals stale build file | Low |
| `buildSrc/` with hardcoded version constants | repo tree | Superseded by version catalogs; invalidation cost | Low |

## Outputs & Where They Feed

| Output | Format | Feeds |
|---|---|---|
| Build facts table (per module: SDKs, features, flavors, plugins) | Table in `docs/factory/audits/DISCOVERY.md` | `PROJECT_STATE.md` Tech Stack Snapshot (P2); all P3 methods |
| Module topology + dependency edges | List in DISCOVERY.md | `02-engineering/02-architecture-mapping.md` diagram |
| Raw dependency dumps (`deps-<config>.txt`) | Saved under `docs/factory/audits/` | `02-engineering/03-dependency-analysis.md` |
| Application-class init list | List in DISCOVERY.md | `02-engineering/04-sdk-inventory.md` |
| Raw permission + component lists | Lists in DISCOVERY.md | Methods `05-service-inventory.md`, `06-manifest-audit.md` |
| Red-flag hits | Pre-findings, promoted to ENG-### in P3 | `docs/factory/audits/ENGINEERING_AUDIT.md` |
| Version stack (Gradle/AGP/Kotlin/JDK) + compatibility verdict | Table in DISCOVERY.md | `02-engineering/migrations/*` ordering decisions (P4) |

Discovery is complete when every row of the build facts table is filled for every module, all seven
reading-order steps have recorded output, and `docs/factory/audits/DISCOVERY.md` exists in the
target repo. Do not begin any audit or migration before that.
