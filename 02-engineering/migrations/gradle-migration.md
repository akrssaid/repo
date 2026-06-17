# Gradle Migration — Upgrading the Gradle Wrapper

Execution-grade guide for upgrading a target app's Gradle wrapper during lifecycle phase **P4 — Modernization**. An agent follows this step-by-step on a real codebase. Gradle is the **first** layer in the mandatory toolchain ordering — every other upgrade depends on it being done correctly.

## Mandatory Global Ordering

All P4 toolchain migrations execute in this exact order, one PR/commit-series per layer:

```text
Gradle → AGP → Kotlin → compileSdk/targetSdk → libraries
```

**Why:** each layer declares minimum requirements on the layer *below* it. AGP requires a minimum Gradle version; the Kotlin Gradle plugin and KSP require minimum AGP/Gradle versions; new compileSdk levels require minimum AGP versions; modern AndroidX/Compose libraries require minimum compileSdk and Kotlin versions. Upgrading out of order produces cascading sync failures that are hard to attribute. Upgrading bottom-up means each step lands on a foundation that already satisfies its requirements. This guide covers the Gradle step only — continue with `02-engineering/migrations/agp-migration.md`, then `kotlin-migration.md`, then `android-sdk-migration.md`.

## Preconditions

| Requirement | Check | If missing |
|---|---|---|
| P3 audit complete | `docs/factory/audits/ENGINEERING_AUDIT.md` exists with current/target versions | Run `01-core/prompts/03-modernization-audit.md` first |
| Project memory current | `docs/factory/PROJECT_STATE.md` has a Tech Stack Snapshot section | Run `01-core/prompts/02-project-memory.md` |
| Clean working tree | `git status --porcelain` is empty | Commit or stash before starting |
| Pre-migration test baseline | Test coverage meets the characterization minimum for a Gradle migration per the migration-type table in `02-engineering/08-testing-strategy.md` | Write the mandated characterization tests **before** migrating — do not assume previously-green tests exist or suffice |
| JDK present | `./gradlew --version` shows JDK 17 (or 11 for legacy AGP 7 builds) | Install JDK 17; set `org.gradle.java.home` in `gradle.properties` if multiple JDKs |
| Current versions known | Read `gradle/wrapper/gradle-wrapper.properties` (`distributionUrl`) and AGP version from root build file or `gradle/libs.versions.toml` | Record both before any change |
| Network access to services.gradle.org | Wrapper download succeeds | Configure proxy in `gradle.properties` (`systemProp.https.proxyHost/Port`) |

## Pre-flight

1. Tag and branch:

```bash
git tag pre-gradle-migration-2026-06-10
git checkout -b chore/gradle-upgrade
```

2. Capture a baseline build (Tier 1–2 per `02-engineering/07-build-verification.md`):

```bash
./gradlew clean assembleDebug --stacktrace 2>&1 | tee docs/factory/reports/baseline-gradle-build.log
./gradlew testDebugUnitTest
```

3. If the baseline fails, **stop**. Fix the baseline or record it as a known failure in `docs/factory/PROJECT_STATE.md` → Known Risks before migrating. Never migrate atop a broken build — you cannot attribute new failures.

## Procedure

### 1. Determine the target Gradle version from AGP compatibility

Gradle is chosen to satisfy the AGP version you will upgrade to **next** (in `agp-migration.md`). Use this matrix:

| AGP version | Minimum Gradle | Recommended Gradle |
|---|---|---|
| 7.4.x | 7.5 | 7.6.x |
| 8.0.x | 8.0 | 8.0.x |
| 8.2.x | 8.2 | 8.2.x |
| 8.5.x | 8.7 | 8.7 |
| 8.7.x | 8.9 | 8.9 |
| 8.9.x | 8.11.1 | 8.11.1+ |

**Verify the current matrix at developer.android.com/build/releases/gradle-plugin before applying** — this table drifts with every AGP release. The June 2026 factory baseline is Gradle 8.13+ (see `08-knowledge/android/version-matrix.md`).

### 2. Plan incremental hops — never jump more than one major version

| Current Gradle | Hop sequence |
|---|---|
| 6.x | 6.9.4 → 7.6.4 → 8.x target |
| 7.x | 7.6.4 → 8.x target |
| 8.x | direct to 8.x target |

Each hop is a separate commit with a passing build. Major-version jumps skip deprecation warnings that became hard errors, leaving you debugging two majors' worth of breakage at once.

### 3. Run the deprecation scan on the CURRENT version first

```bash
./gradlew build --warning-mode all 2>&1 | tee docs/factory/reports/gradle-deprecations.log
```

Triage the output:

- **"Deprecated Gradle features were used... incompatible with Gradle X"** — these break on the next major. Fix each before hopping: typical culprits are `compile`/`testCompile` configurations (→ `implementation`/`testImplementation`), `jcenter()` (→ `mavenCentral()`), `task taskName << { }` syntax (→ `tasks.register`), and old third-party plugins (upgrade or remove).
- Deprecations inside third-party plugin code you cannot edit: upgrade the plugin; if no compatible version exists, record it as a blocker in `docs/factory/PROJECT_STATE.md` → Known Risks and stop the hop at the last compatible Gradle.

### 4. Upgrade the wrapper

Always use the wrapper task — never hand-edit only the properties file (the task also updates wrapper scripts and the wrapper JAR):

```bash
./gradlew wrapper --gradle-version 7.6.4 --distribution-type bin
# next hop, after a green build:
./gradlew wrapper --gradle-version 8.13 --distribution-type bin
```

Verify the result in `gradle/wrapper/gradle-wrapper.properties`:

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
```

Confirm:

```bash
./gradlew --version
```

Commit each hop: `git commit -am "chore: gradle wrapper 7.6.4 -> 8.13"` — include `gradlew`, `gradlew.bat`, `gradle/wrapper/*`.

### 5. Fix per-hop breakage

After each hop, run `./gradlew assembleDebug --stacktrace`. Common Gradle 8 failures:

- `Could not find method compile()` — replace with `implementation`.
- `Cannot run Gradle on JVM 11` style errors — Gradle 8.x wants JDK 17; set it in `gradle.properties`:

```properties
org.gradle.java.home=C\:\\Program Files\\Java\\jdk-17
```

(Prefer the JDK toolchain DSL once AGP is on 8.x — see `agp-migration.md` step 4.)
- Tasks that read project state at execution time fail under stricter validation — annotate inputs/outputs or upgrade the offending plugin.

### 6. Centralize repository and plugin management in settings (Gradle 7+ pattern)

If the project still declares repositories per module or resolves the AGP classpath via `buildscript {}`, modernize `settings.gradle(.kts)` now — version catalogs (step 8) and many plugins assume this layout:

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "target-app"
include(":app")
```

Then delete every `repositories {}` block from module build files and the root `allprojects {}` block. If a legacy plugin is only resolvable via `buildscript { classpath(...) }`, keep that one entry and record it as debt — everything else moves to the `plugins {}` DSL.

### 7. Groovy → Kotlin DSL conversion (optional)

Convert when: the project will live for years, build logic is non-trivial, or you are adopting version catalogs anyway. Skip when: the app is in maintenance mode and build files are trivial — conversion is churn without payoff.

Procedure (one file per commit): rename `settings.gradle` → `settings.gradle.kts`, then root `build.gradle` → `build.gradle.kts`, then each module. Mechanical changes:

```diff
-apply plugin: 'com.android.application'
+plugins { id("com.android.application") }

-compileSdkVersion 34
+compileSdk = 34

-buildConfigField "String", "API_URL", '"https://api.example.com"'
+buildConfigField("String", "API_URL", "\"https://api.example.com\"")

-release {
-    minifyEnabled true
+release {
+    isMinifyEnabled = true
```

Single quotes → double quotes, `=` for property assignment, `is` prefix for booleans, explicit parentheses for method calls. Build after every file.

### 8. Adopt a version catalog

Create `gradle/libs.versions.toml`, recording the project's **current** versions — catalog adoption is a refactor at the Gradle layer, not a version upgrade:

```toml
[versions]
kotlin = "1.9.24"      # whatever the project uses TODAY; bumped later in kotlin-migration.md
coreKtx = "1.16.0"
junit = "4.13.2"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
junit = { group = "junit", name = "junit", version.ref = "junit" }

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

The `agp` version pin and `android-application`/`android-library` plugin aliases are deliberately absent here — AGP pinning is owned by `agp-migration.md` step 2, which adds them when that layer migrates. Do not pre-pin AGP in this step.

Migrate **module by module**, one commit each:

```diff
-implementation "androidx.core:core-ktx:1.16.0"
+implementation(libs.androidx.core.ktx)
```

```diff
 plugins {
-    id 'org.jetbrains.kotlin.android' version '1.9.24' apply false
+    alias(libs.plugins.kotlin.android) apply false
 }
```

Catalog accessor names: dots in TOML keys become dots in accessors; hyphens become dots (`core-ktx` → `libs.core.ktx`). Sync after each module. The catalog becomes the single source of truth consumed by all later migration guides.

### 9. Enable build cache and (cautiously) configuration cache

In `gradle.properties`:

```properties
org.gradle.caching=true
org.gradle.parallel=true
# Enable only after the test run below passes:
org.gradle.configuration-cache=true
org.gradle.configuration-cache.problems=warn
```

Test configuration cache compatibility before committing:

```bash
./gradlew assembleDebug --configuration-cache 2>&1 | tee docs/factory/reports/config-cache-probe.log
```

**Risk notes:** configuration cache breaks plugins that access `Project` at execution time — google-services, Crashlytics, and Hilt are compatible in current versions, but older versions and bespoke build scripts are not. If the probe reports problems in code you don't control, leave configuration cache **off** and record the blocker; `problems=warn` is for diagnosis only, never ship it silently ignoring failures. Build cache is lower risk but can mask stale-output bugs in misbehaving custom tasks — when chasing a weird build issue, retest with `--no-build-cache` once.

## Verification

Run the canonical tiers from `02-engineering/07-build-verification.md` after the final hop. Toolchain migrations always require Tier 3 (see its mandatory-tier table):

| Tier | Command | Pass criterion |
|---|---|---|
| 1 — Fast | `./gradlew help` then `./gradlew compileDebugKotlin compileDebugJavaWithJavac` | Configures and compiles with zero errors |
| 2 — Standard | `./gradlew clean assembleDebug :app:lintDebug testDebugUnitTest` | BUILD SUCCESSFUL; APK produced; no new lint errors vs baseline; characterization tests (per `02-engineering/08-testing-strategy.md`) pass |
| 3 — Release-grade | `./gradlew clean bundleRelease assembleRelease`, then install the release build on device/emulator and run the smoke procedure from `07-build-verification.md` | Bundle + non-trivial `mapping.txt` produced; app launches; smoke flows clean; no FATAL in logcat |

Also rerun `./gradlew build --warning-mode all` and confirm the deprecation count did not grow. Diff the new log against `docs/factory/reports/gradle-deprecations.log`.

## Rollback

Per-hop (preferred — hops are individual commits):

```bash
git revert <hop-commit-sha>
```

Full rollback to pre-migration state:

```bash
git checkout main
git branch -D chore/gradle-upgrade
git checkout pre-gradle-migration-2026-06-10 -- gradle/ gradlew gradlew.bat   # only if main was contaminated
```

After any rollback: `./gradlew --stop` (kill stale daemons running the new version), delete `.gradle/` in the project root if sync remains confused, rebuild, and record the failure cause in `docs/factory/PROJECT_STATE.md` → Known Risks.

## Gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| Hand-edited `distributionUrl` only | Wrapper JAR/scripts stale; CI behaves differently than local | Always run the `wrapper` task; commit all four wrapper files |
| Stale Gradle daemon | New version "not taking effect", bizarre cache errors | `./gradlew --stop`, then rebuild |
| `-all` vs `-bin` distribution | Slow first sync, larger download | Use `--distribution-type bin`; IDEs no longer need `-all` for sources |
| Gradle 8 + old JDK | `Unsupported class file major version` / toolchain errors | JDK 17 for Gradle 8.x; pin via `org.gradle.java.home` or toolchains |
| `jcenter()` still in repositories | Dependencies resolve flakily or 403 | Replace with `mavenCentral()`; find leftovers with `grep -r jcenter --include=*.gradle*` |
| Repositories declared in both `settings.gradle` and module files | `Build was configured to prefer settings repositories` error | Keep `dependencyResolutionManagement` in settings; delete module-level `repositories {}` |
| Configuration cache + google-services (old) | `invocation of 'Task.project' at execution time is unsupported` | Upgrade plugin; else disable configuration cache |
| OneDrive/synced project folder (Windows) | Random file-lock build failures | Pause sync during builds or move repo out of OneDrive; at minimum exclude `build/` and `.gradle/` |
| Version catalog accessor mismatch | `Could not get unknown property 'libs'` | Catalog must be at `gradle/libs.versions.toml`; Gradle ≥7.4; check key→accessor hyphen mapping |
| CI uses its own Gradle install | CI green locally-red (or inverse) | CI must invoke `./gradlew` (the wrapper), never a system Gradle |

## Documentation Updates

On completion, update `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot (Gradle old → new, date 2026-06-10, distribution type, caches enabled y/n) and append a `docs/factory/CHANGELOG.md` entry. Then proceed to `02-engineering/migrations/agp-migration.md`.
