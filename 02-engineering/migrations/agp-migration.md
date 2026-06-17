# AGP Migration — Upgrading the Android Gradle Plugin

Execution-grade guide for upgrading the Android Gradle Plugin (AGP) during lifecycle phase **P4 — Modernization**. An agent follows this step-by-step on a real codebase. AGP is the **second** layer in the toolchain ordering and carries the highest breaking-change density of any layer — especially the 7→8 boundary.

## Mandatory Global Ordering

```text
Gradle → AGP → Kotlin → compileSdk/targetSdk → libraries
```

**Why:** each layer declares minimum requirements on the layer below. AGP requires a minimum Gradle version (so Gradle goes first — see `02-engineering/migrations/gradle-migration.md`); the Kotlin plugin and KSP are tested against specific AGP ranges (so AGP precedes Kotlin); new compileSdk levels require minimum AGP versions — compileSdk 36 requires AGP **8.9.1** minimum (full compileSdk↔AGP pairing table in `android-sdk-migration.md` step 1; canonical version matrix in `08-knowledge/android/version-matrix.md`); libraries require all of the above. This guide assumes the Gradle step is complete and green.

## Preconditions

| Requirement | Check | If missing |
|---|---|---|
| Gradle migration done | Wrapper at the version required by target AGP (see matrix in `gradle-migration.md` step 1) | Complete `gradle-migration.md` first |
| Clean tree, green build | `git status --porcelain` empty; `./gradlew assembleDebug` passes | Fix or record before starting |
| Pre-migration test baseline | Test coverage meets the characterization minimum for an AGP migration per the migration-type table in `02-engineering/08-testing-strategy.md` | Write the mandated characterization tests **before** bumping — do not assume previously-green tests exist or suffice |
| JDK 17 available | `java -version` or `./gradlew --version` | **AGP 8.0+ requires JDK 17 to run Gradle.** Install it; this is non-negotiable for AGP 8 |
| Current AGP known | Root build file or `gradle/libs.versions.toml` `[versions] agp` | Record in `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot |
| Release signing config available | Can run `assembleRelease` locally (or `minifyEnabled` debug variant exists) | R8 verification (step 7) is mandatory; arrange a minified build path |
| Target version chosen | Per `docs/factory/audits/ENGINEERING_AUDIT.md`; factory baseline AGP 8.9+ | Verify latest stable at developer.android.com/build/releases/gradle-plugin |

## Pre-flight

```bash
git tag pre-agp-migration-2026-06-10
git checkout -b chore/agp-upgrade
./gradlew clean assembleDebug --stacktrace 2>&1 | tee docs/factory/reports/baseline-agp-build.log
./gradlew testDebugUnitTest
```

Additionally snapshot the current release mapping for later diffing (if a release build exists):

```bash
./gradlew assembleRelease
cp app/build/outputs/mapping/release/mapping.txt docs/factory/reports/baseline-mapping.txt
```

## Procedure

### 1. Plan hops

Never jump more than one AGP major per hop. From 7.x: go to the latest 7.4.x first, fix all warnings, then cross to 8.x. Within 8.x, hop through minors as needed by the Gradle version you already installed (e.g., 8.2 → 8.7 → 8.9). Each hop = one commit with a green Tier-2 build.

### 2. Bump the version

```diff
# gradle/libs.versions.toml
 [versions]
-agp = "7.4.2"
+agp = "8.9.2"
```

or in legacy root `build.gradle`:

```diff
-        classpath 'com.android.tools.build:gradle:7.4.2'
+        classpath 'com.android.tools.build:gradle:8.9.2'
```

Sync (`./gradlew help`) and collect errors. Android Studio's AGP Upgrade Assistant automates steps 3–5 below; this guide is the manual equivalent so an agent can do it headlessly.

### 3. AGP 7 → 8 breaking changes (apply each)

| Change | Old behavior | AGP 8 behavior | Required action |
|---|---|---|---|
| `namespace` | `package` attr in `AndroidManifest.xml` defined R class package | `package` attr forbidden; `namespace` required in build file | Step 3a |
| `buildConfig` | `BuildConfig` generated always | **Off by default** | Step 3b |
| Non-transitive R classes | Module R contained all transitive resources | `android.nonTransitiveRClass=true` is default | Step 3c |
| R8 full mode | Compat mode (lenient keep semantics) | **Full mode default** | Step 3d + step 7 |
| `android:exported` | Implicit for components with intent-filters | Must be explicit (enforced since targetSdk 31; AGP 8 manifest merger errors) | Step 3e |
| JDK | JDK 11 to run builds | **JDK 17 required** | Step 4 |
| `flavorDimensions`/options as properties | Method syntax | Property assignment in DSL | Mechanical sync-error fixes |

**3a. Namespace.** For every module:

```diff
# app/src/main/AndroidManifest.xml
-<manifest xmlns:android="http://schemas.android.com/apk/res/android"
-    package="com.example.app">
+<manifest xmlns:android="http://schemas.android.com/apk/res/android">
```

```diff
# app/build.gradle.kts
 android {
+    namespace = "com.example.app"
```

If the manifest `package` differed from `applicationId`, set `namespace` to the old `package` value (it controls R class and code packages), keep `applicationId` as-is. Fix imports of `R` and `BuildConfig` that referenced the wrong package.

**3b. BuildConfig.** If any code references `BuildConfig` (search: `grep -rn "BuildConfig\." app/src --include=*.kt --include=*.java`):

```kotlin
// app/build.gradle.kts
android {
    buildFeatures {
        buildConfig = true
    }
}
```

Otherwise leave it off (faster builds).

**3c. Non-transitive R.** Symptom: unresolved `R.` references for resources defined in other modules. Fix properly by importing the owning module's R class (`import com.example.library.R as libR`). Temporary escape hatch (record as debt in `docs/factory/PROJECT_STATE.md` → Backlog):

```properties
# gradle.properties
android.nonTransitiveRClass=false
```

**3d. R8 full mode default.** Full mode is more aggressive: it no longer keeps default constructors of classes reached only via reflection, strips more attributes, and optimizes across feature boundaries. Crashes appear **only in release builds at runtime** — see step 7 for mandatory verification. Typical keep-rule additions in `proguard-rules.pro`:

```proguard
# Gson/reflection-based serialization models
-keepclassmembers class com.example.app.model.** { <init>(); <fields>; }
# Classes loaded by name from manifest/XML
-keep class com.example.app.MyCustomView { <init>(android.content.Context, android.util.AttributeSet); }
```

Last-resort compat fallback (record as debt in `docs/factory/PROJECT_STATE.md` → Backlog): `android.enableR8.fullMode=false` in `gradle.properties`.

**3e. Explicit exported.** For every `<activity>`, `<service>`, `<receiver>` with an `<intent-filter>`:

```diff
-        <activity android:name=".ShareActivity">
+        <activity android:name=".ShareActivity"
+            android:exported="true">
             <intent-filter>
```

Rule: launcher activities and components that must receive external intents → `exported="true"`; everything else → `exported="false"`. Check the merged manifest (obtain it per `02-engineering/06-manifest-audit.md` Step 0) for library-contributed components needing overrides via `tools:node="merge"`.

**3f. Renamed/removed DSL blocks.** AGP 8 removed long-deprecated DSL names. Fix per sync error — the error text names the replacement:

| Removed (AGP 7) | Replacement (AGP 8) |
|---|---|
| `packagingOptions { }` | `packaging { }` |
| `aaptOptions { }` | `androidResources { }` |
| `lintOptions { }` | `lint { }` |
| `dexOptions { }` | Deleted — no replacement needed (D8 manages this) |
| `jacoco { }` (AGP block) | `testCoverage { }` |
| `flavorDimensions "x"` (method) | `flavorDimensions += "x"` (property) |
| `targetSdkVersion` in library modules | Removed — use `lint.targetSdk` for lint checks; targetSdk is app-module-only |

### 4. JDK 17 and jvmTarget alignment

AGP 8 runs the build on JDK 17, but **source/target compatibility of your code is separate** and must agree between Java and Kotlin compilation:

```kotlin
// each module's build.gradle.kts
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
kotlin {
    compilerOptions {
        jvmTarget.set(org.jetbrains.kotlin.gradle.dsl.JvmTarget.JVM_17)
    }
}
```

Mismatch symptom: `Inconsistent JVM-target compatibility detected for tasks 'compileDebugJavaWithJavac' (17) and 'compileDebugKotlin' (1.8)`. If a library module must stay on 1.8 bytecode for consumers, set **both** to 1.8 consistently — never mixed. Prefer pinning the build JVM via toolchains so CI and local agree:

```kotlin
kotlin { jvmToolchain(17) }
```

### 5. Companion plugin compatibility

These plugins are version-paired with AGP/Kotlin and break silently if stale. Upgrade them in the same hop:

| Plugin | Pairing rule | June 2026 known-good (verify current) |
|---|---|---|
| KSP (`com.google.devtools.ksp`) | Version is `<kotlinVersion>-<kspVersion>`; **must** match Kotlin exactly | `2.1.20-1.0.31` with Kotlin 2.1.20 |
| Hilt (`com.google.dagger.hilt.android`) | ≥2.44 for AGP 8; keep plugin and runtime same version | 2.56+ |
| google-services | ≥4.3.15 for AGP 8 | 4.4.x |
| Crashlytics gradle | ≥2.9.x for AGP 8 | 3.x |
| Navigation safe-args | Match Navigation runtime; ≥2.5.3 for AGP 8 | 2.8.x+ |

Check each in `gradle/libs.versions.toml` and bump together. If KSP isn't adopted yet, that happens in `kotlin-migration.md` — do not introduce it here.

### 6. 8.x notables on the way to 8.9+

- **Kotlin DSL is the default** for new modules since Studio Giraffe/AGP 8.1 — existing Groovy files keep working; conversion guidance lives in `gradle-migration.md` step 7.
- **AGP 8.9+ built-in Kotlin support**: `com.android.application` can compile Kotlin without the separate `org.jetbrains.kotlin.android` plugin in some configurations. **Keep the explicit Kotlin plugin** for now — KSP, Compose, and most ecosystem tooling still key off it; revisit when the ecosystem settles.
- Minor 8.x hops occasionally tighten manifest merger and lint; treat new lint errors as Tier 2 verification failures (per `02-engineering/07-build-verification.md`) and fix rather than baseline-suppress, unless volume is unmanageable (`./gradlew updateLintBaseline` then schedule the debt).

### 7. ProGuard/R8 verification — mandatory, this is where silent breakage lives

R8 full-mode breakage does not appear in debug builds, unit tests, or even `assembleRelease` success — only at runtime on stripped code paths.

The substeps below are the **gate summary** — the minimum proof this migration may not skip. The deep method (keep-rule authoring patterns, `mapping.txt`/`usage.txt` forensics, shrink-baseline comparison, resource shrinking) is `02-engineering/09-r8-shrinking.md` — switch to it when this gate fails or the app's keep rules need real work.

1. Build release:

```bash
./gradlew assembleRelease
```

2. Confirm shrinking ran and inspect outputs: `app/build/outputs/mapping/release/` must contain `mapping.txt`, `usage.txt` (removed code), `seeds.txt` (kept by rules).
3. Search `usage.txt` for classes you know are reflection-reached (serialization models, `Class.forName` targets, JNI-called classes). If present there, they were **removed** — add keep rules.
4. Decode and inspect the artifact:

```bash
# APK Analyzer CLI — confirm expected classes/obfuscation level
"$ANDROID_HOME/cmdline-tools/latest/bin/apkanalyzer" dex packages app/build/outputs/apk/release/app-release.apk | head -50
```

```powershell
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\apkanalyzer.bat" dex packages app\build\outputs\apk\release\app-release.apk | Select-Object -First 50
```

5. Install the obfuscated build on a real device/emulator and exercise: app launch, every serialization boundary (network responses parsing), deep links, WebView JS bridges, any reflection-based feature. Watch logcat:

```bash
adb install -r app/build/outputs/apk/release/app-release.apk
adb logcat -s AndroidRuntime:E
```

6. Any `ClassNotFoundException`, `NoSuchMethodException`, missing-field deserialization (silent nulls!) → add targeted keep rules, rebuild, retest. Diff `mapping.txt` size against `docs/factory/reports/baseline-mapping.txt` for sanity (full mode typically shrinks more).
7. Archive the new `mapping.txt` reference in `docs/factory/reports/`.

## Verification

Run the canonical tiers from `02-engineering/07-build-verification.md`. Toolchain migrations always require Tier 3 (see its mandatory-tier table):

| Tier | Command | Pass criterion |
|---|---|---|
| 1 — Fast | `./gradlew help` then `./gradlew compileDebugKotlin compileDebugJavaWithJavac` | No configuration or compile errors |
| 2 — Standard | `./gradlew clean assembleDebug :app:lintDebug testDebugUnitTest` | BUILD SUCCESSFUL; no new lint errors vs baseline; characterization tests (per `02-engineering/08-testing-strategy.md`) pass |
| 3 — Release-grade | `./gradlew clean bundleRelease assembleRelease` + step 7 above in full; smoke-test debug **and** release builds on device | Minified build verified on device; core flows work in both |

Tier 3 is **not optional** for AGP 7→8 because of R8 full mode — debug-only verification proves nothing about the shipped artifact.

## Rollback

```bash
# Whole-layer rollback
git checkout main && git branch -D chore/agp-upgrade
# Or revert a single hop
git revert <hop-commit-sha>
./gradlew --stop && ./gradlew clean
```

If only R8 full mode is broken and release is urgent: ship with `android.enableR8.fullMode=false` and file a High-severity item in `docs/factory/PROJECT_STATE.md` → Backlog to fix keep rules properly (method: `02-engineering/09-r8-shrinking.md`). Do not roll back the entire AGP upgrade for an R8-rules-only problem.

## Gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| JDK 17 missing/wrong | `Android Gradle plugin requires Java 17 to run` | Install JDK 17; set `org.gradle.java.home` or Studio Gradle JDK; check CI image too |
| Namespace vs applicationId confusion | `Unresolved reference: R` / wrong `BuildConfig` import after 3a | `namespace` = old manifest `package`; `applicationId` unchanged |
| Library manifests still have `package` | `Incorrect package="..." found in source AndroidManifest.xml` | Remove the attr in **every** module and remote-source AARs you control |
| BuildConfig fields vanish | `Unresolved reference: BuildConfig` | `buildFeatures { buildConfig = true }` per module that uses it |
| R8 full mode + Gson | Fields silently null after deserialization in release only | Keep rules on model packages (step 3d); prefer migrating to kotlinx.serialization/Moshi codegen later |
| `android:exported` merger error | `android:exported needs to be explicitly specified` at manifest merge | Step 3e; for library-owned components add explicit override in app manifest |
| jvmTarget mismatch | `Inconsistent JVM-target compatibility` | Align compileOptions + kotlin jvmTarget in every module (step 4) |
| Stale KSP/Hilt with new AGP | `ClassCastException` inside plugin during configuration, or generated code missing | Pair plugin versions per step 5 table |
| Lint runs on JDK 17 bytecode rules | New lint errors after upgrade only | Fix, or regenerate baseline consciously and record debt |
| `aaptOptions`→`androidResources` and other renamed DSL blocks | `Could not find method aaptOptions()` | Rename per sync error message; AGP error text names the replacement |
| Firebase Perf / older exotic plugins | Build hangs or variant API crashes | Upgrade to AGP-8-compatible version; if none exists, remove and record blocker |
| `usage.txt` not consulted before device test | Hours lost bisecting a release-only crash | Step 7.3 first — grep `usage.txt` for the crashing class; it tells you immediately if R8 removed it |
| Debug-only verification of R8 changes | "Everything passes" but release users crash | Debug builds don't minify by default; Tier 3 + step 7 on the release artifact are the only valid proof |
| `mapping.txt` not archived per release | Play crash reports unreadable after ship | Upload mapping with the release (Play does this for AABs); archive a copy in `docs/factory/reports/` per `10-releases/release-workflow.md` |

## Documentation Updates

Update `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot (AGP old → new, JDK 17 confirmed, R8 full-mode status, plugin pairings) and append a `docs/factory/CHANGELOG.md` entry dated 2026-06-10. Then proceed to `02-engineering/migrations/kotlin-migration.md`.
