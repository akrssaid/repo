# Kotlin Migration — Upgrading Kotlin and the Compiler-Coupled Ecosystem

Execution-grade guide for upgrading Kotlin (including the K2 compiler crossing at 2.0), migrating kapt→KSP, adopting the new Compose compiler plugin, and removing legacy synthetics — during lifecycle phase **P4 — Modernization**. An agent follows this step-by-step on a real codebase. Kotlin drags an ecosystem with it: KSP, Compose compiler, coroutines, and serialization are all version-coupled to the compiler.

## Mandatory Global Ordering

```text
Gradle → AGP → Kotlin → compileSdk/targetSdk → libraries
```

**Why:** each layer's minimum requirements sit on the layer below. The Kotlin Gradle plugin requires minimum Gradle/AGP versions (so those go first — see `02-engineering/migrations/gradle-migration.md` and `agp-migration.md`); KSP versions are lockstep with Kotlin; the Compose compiler is shipped with Kotlin since 2.0; modern AndroidX libraries then require minimum Kotlin metadata versions (so libraries come last). This guide assumes Gradle and AGP migrations are complete and green.

## Preconditions

| Requirement | Check | If missing |
|---|---|---|
| Gradle + AGP migrations done | Wrapper and AGP at targets from `docs/factory/audits/ENGINEERING_AUDIT.md`; baseline green | Complete those guides first |
| Clean tree | `git status --porcelain` empty | Commit/stash |
| Pre-migration test baseline | Test coverage meets the characterization minimum for a Kotlin/K2 + kapt→KSP migration per the migration-type table in `02-engineering/08-testing-strategy.md` (DAO, DI-graph, and serialization round-trip coverage especially) | Write the mandated characterization tests **before** the first hop — do not assume previously-green tests exist or suffice |
| Current Kotlin known | `[versions] kotlin` in `gradle/libs.versions.toml` or root build file | Record it |
| Annotation processors inventoried | `grep -rn "kapt\|ksp" --include=*.gradle* --include=*.toml .` lists Room/Hilt/Glide/Moshi/etc. | Build the inventory now — it drives step 4 |
| Compose usage known | `buildFeatures { compose = true }` present? `composeOptions` block? | Determines whether step 5 applies |
| Synthetics usage known | `grep -rn "kotlinx.android.synthetic" app/src` | Determines whether step 6 applies |

## Pre-flight

```bash
git tag pre-kotlin-migration-2026-06-10
git checkout -b chore/kotlin-upgrade
./gradlew clean assembleDebug testDebugUnitTest --stacktrace 2>&1 | tee docs/factory/reports/baseline-kotlin-build.log
```

Record current kapt processor list and annotation-processing build time (`./gradlew assembleDebug --profile`) — you will use it to prove the KSP win.

## Procedure

### 1. Hop plan: 1.8 → 1.9 → 2.0 (K2) → 2.1+

| Hop | Risk | Notes |
|---|---|---|
| 1.8.x → 1.9.24 | Low | Mostly deprecation surfacing; do the deprecated-API sweep (step 8) here |
| 1.9.24 → 2.0.x | **High** | K2 compiler crossing + Compose compiler plugin change (step 5). Do kapt→KSP (step 4) **before** this hop — KSP2 and K2 behave better together, and you remove kapt's K2 friction |
| 2.0.x → 2.1.x | Low–Medium | New language features; few breaks. Factory baseline: Kotlin 2.1+ (verify latest stable at kotlinlang.org/docs/releases.html) |

One hop per commit, green build between hops. Bump in the catalog:

```diff
 [versions]
-kotlin = "1.9.24"
+kotlin = "2.1.20"
-ksp = "1.9.24-1.0.20"
+ksp = "2.1.20-1.0.31"
```

**KSP version must always be `<exact-kotlin-version>-<ksp-revision>`.** Mismatched pairs fail with confusing `language version` errors.

### 2. Pre-cross K2 probe (while still on 1.9)

Compile with K2 enabled to surface breakage before committing to 2.0:

```bash
./gradlew clean assembleDebug -Pkotlin.experimental.tryK2=true 2>&1 | tee docs/factory/reports/k2-probe.log
```

Triage every new error using the table in step 3. Fix them on 1.9 (the fixes are backward-compatible), then do the 2.0 hop.

### 3. K2 migration specifics — what typically breaks

| Category | Old (K1) behavior | K2 behavior | Fix pattern |
|---|---|---|---|
| Smart casts from open/custom-getter properties | Sometimes (unsoundly) smart-cast | Refused: `Smart cast to 'X' is impossible` | Capture to a local: `val s = this.state; if (s is Loaded) { s.data }` |
| Smart casts — improved cases | Not propagated from `||`, inline lambdas, local vars | K2 smart-casts **more** in safe cases | Usually no action; occasionally exposes now-redundant casts (lint warns) |
| Builder inference | Lenient postponed-type resolution | Stricter; `Cannot infer type` in `buildList { }` / `flow { }` chains | Add explicit type args: `buildList<Item> { ... }` |
| Property vs function resolution ambiguity | Resolved by K1 quirk order | Resolution order corrected | Qualify explicitly; rare |
| Exhaustive `when` over sealed/enum from binary deps | Sometimes missed | Properly enforced | Add missing branches or `else` |
| `INVISIBLE_REFERENCE` suppressions | Worked | Hard error — internal API access across modules blocked | Stop using internals; expose proper API |
| Annotation processing via kapt | kapt on K1 frontend | kapt runs in K1 fallback mode under 2.0 (slow, deprecated) | Migrate to KSP first (step 4) |

After the 2.0 hop, also set explicit language/api versions only if you must hold back temporarily:

```kotlin
kotlin { compilerOptions { languageVersion.set(KotlinVersion.KOTLIN_1_9) } }  // escape hatch only; record as debt
```

### 4. kapt → KSP migration, per processor

Add the KSP plugin once:

```diff
 # libs.versions.toml [plugins]
+ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

```diff
 // app/build.gradle.kts plugins
-    id("org.jetbrains.kotlin.kapt")
+    alias(libs.plugins.ksp)
```

Migrate one processor per commit, building between each. Each processor flip must meet the kapt→KSP per-migration test minimums in `02-engineering/08-testing-strategy.md` — run the relevant slice (Room DAO tests, Hilt graph compile + injection smoke, Moshi/serialization round-trips) before committing that processor's commit, not just at the end.

**Room** (≥2.5.0 for KSP; use 2.6+):

```diff
-    kapt("androidx.room:room-compiler:2.6.1")
+    ksp("androidx.room:room-compiler:2.6.1")
```

Room schema location moves from kapt arguments to the KSP arg (this is the canonical schema-export form — `02-engineering/11-data-migration-safety.md` depends on these JSON schemas for migration testing; losing the export silently breaks that safety net):

```diff
-    defaultConfig { javaCompileOptions { annotationProcessorOptions { arguments += mapOf("room.schemaLocation" to "$projectDir/schemas") } } }
+    ksp { arg("room.schemaLocation", "$projectDir/schemas") }
```

**Hilt** (≥2.48 for KSP; pair with current 2.56+):

```diff
-    kapt("com.google.dagger:hilt-compiler:2.56")
+    ksp("com.google.dagger:hilt-compiler:2.56")
-    kapt("androidx.hilt:hilt-compiler:1.2.0")
+    ksp("androidx.hilt:hilt-compiler:1.2.0")
```

**Glide** (≥4.14 ships a KSP processor):

```diff
-    kapt("com.github.bumptech.glide:compiler:4.16.0")
+    ksp("com.github.bumptech.glide:ksp:4.16.0")
```

Note the **artifact name changes** (`compiler` → `ksp`).

**Moshi** (codegen ≥1.13 supports KSP):

```diff
-    kapt("com.squareup.moshi:moshi-kotlin-codegen:1.15.1")
+    ksp("com.squareup.moshi:moshi-kotlin-codegen:1.15.1")
```

When the last processor is migrated, remove the kapt plugin and any `kapt {}` blocks. Processors with **no KSP support** (legacy Dagger-android extensions, some niche libraries): keep kapt for that module only, record High-severity debt in `docs/factory/PROJECT_STATE.md` → Backlog, and plan replacement — kapt is deprecated and frozen.

Verify generated code parity: `app/build/generated/ksp/debug/` must contain the equivalents of what `app/build/generated/source/kapt/` had. Run the app — Hilt graph errors surface at compile time, Room schema mismatches at first DB open.

### 5. Compose compiler — versioned with Kotlin since 2.0

Before 2.0 you pinned a `kotlinCompilerExtensionVersion` against a compatibility table. From Kotlin 2.0, the Compose compiler ships **with Kotlin** and is applied as a Gradle plugin. Exact diff:

```diff
 # libs.versions.toml
 [plugins]
+compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
```

```diff
 // app/build.gradle.kts
 plugins {
     alias(libs.plugins.android.application)
     alias(libs.plugins.kotlin.android)
+    alias(libs.plugins.compose.compiler)
 }

 android {
     buildFeatures {
         compose = true
     }
-    composeOptions {
-        kotlinCompilerExtensionVersion = "1.5.14"
-    }
 }
```

The plugin version **is** the Kotlin version — one less compatibility table forever. Compiler options (stability config, reports) move to the `composeCompiler {}` block:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_reports")
}
```

Apply this in the **same commit** as the 2.0 hop — the build fails without it when Compose is enabled.

### 6. kotlin-android-extensions / synthetics removal → ViewBinding

The `kotlin-android-extensions` plugin is gone entirely in Kotlin 1.8+ (Gradle plugin removed in 1.9 line). If `grep -rn "kotlinx.android.synthetic" app/src` hits:

1. Enable ViewBinding:

```kotlin
android { buildFeatures { viewBinding = true } }
```

2. Remove the plugin:

```diff
-    id("kotlin-android-extensions")
```

3. Convert per screen (one commit per logical group):

```diff
-import kotlinx.android.synthetic.main.activity_main.*
+import com.example.app.databinding.ActivityMainBinding

 class MainActivity : AppCompatActivity() {
+    private lateinit var binding: ActivityMainBinding
     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)
-        setContentView(R.layout.activity_main)
-        titleText.text = "Hello"
+        binding = ActivityMainBinding.inflate(layoutInflater)
+        setContentView(binding.root)
+        binding.titleText.text = "Hello"
     }
 }
```

Fragments: inflate in `onCreateView`, null the binding reference in `onDestroyView`. `@Parcelize` users: the annotation moved to the still-supported `kotlin-parcelize` plugin — add `id("kotlin-parcelize")` and change imports to `kotlinx.parcelize.Parcelize`.

### 7. Coroutines / serialization pairing

Bump in the same layer (they are compiler-metadata-coupled):

| Kotlin | kotlinx-coroutines | kotlinx-serialization runtime |
|---|---|---|
| 1.8.x | 1.6.4–1.7.x | 1.5.x |
| 1.9.x | 1.7.x–1.8.x | 1.6.x |
| 2.0.x | 1.8.x–1.9.x | 1.6.3–1.7.x |
| 2.1.x | 1.9.x–1.10.x | 1.7.x–1.8.x |

Verify current pairings at github.com/Kotlin/kotlinx.coroutines and kotlinx.serialization releases. The serialization **plugin** (`org.jetbrains.kotlin.plugin.serialization`) version must equal the Kotlin version exactly. Symptom of stale metadata: `Module was compiled with an incompatible version of Kotlin. The binary version of its metadata is X, expected Y` — bump the named library.

### 8. Deprecated stdlib API sweep

On the final version, compile with warnings surfaced and fix:

```bash
./gradlew compileDebugKotlin 2>&1 | grep -i "deprecat" | sort -u > docs/factory/reports/kotlin-deprecations.txt
```

```powershell
./gradlew compileDebugKotlin | Select-String -Pattern "deprecat" | Sort-Object -Unique | Out-File -Encoding utf8 docs\factory\reports\kotlin-deprecations.txt
```

Common sweep items: `toUpperCase()`/`toLowerCase()` → `uppercase()`/`lowercase()`; `capitalize()` → `replaceFirstChar { it.uppercase() }`; `String.format` locale warnings → pass `Locale.ROOT` or use templates; `kotlin-stdlib-jdk7/jdk8` artifacts → plain `kotlin-stdlib` (merged since 1.8); `Date().time` patterns → `kotlinx-datetime` or `java.time` (minSdk 26+ has it natively). Fix all errors; fix warnings opportunistically and record the remainder as Low-severity debt.

## Verification

Run the canonical tiers from `02-engineering/07-build-verification.md`. Toolchain migrations always require Tier 3 (see its mandatory-tier table):

| Tier | Command | Pass criterion |
|---|---|---|
| 1 — Fast | `./gradlew help` then `./gradlew compileDebugKotlin compileDebugJavaWithJavac` | No errors; KSP/Kotlin versions resolve; K2 compiles clean |
| 2 — Standard | `./gradlew clean assembleDebug :app:lintDebug testDebugUnitTest` | BUILD SUCCESSFUL with K2; all tests pass — DI/DB/serialization characterization tests (per `02-engineering/08-testing-strategy.md`) especially, since they cover KSP-generated code paths |
| 3 — Release-grade | `./gradlew clean bundleRelease assembleRelease` (re-check mapping per `agp-migration.md` step 7); device smoke test of debug + release builds | R8 handles new Kotlin metadata; Hilt injection, Room queries, JSON parsing, all Compose screens render in both builds |

Extra Kotlin-specific checks: compare KSP-generated output exists for every formerly-kapt processor; confirm annotation-processing time improved vs the pre-flight `--profile` snapshot; run instrumented tests if Room migrations exist (`./gradlew connectedDebugAndroidTest`).

## Rollback

```bash
git revert <hop-commit-sha>          # single hop or single processor migration
git checkout main && git branch -D chore/kotlin-upgrade   # full layer
./gradlew --stop && ./gradlew clean
```

The kapt→KSP commits are independently revertible per processor — if only Glide-KSP misbehaves, revert that one commit and keep the rest. If K2 itself blocks you on 2.0+, the `languageVersion 1.9` escape hatch (step 3) buys time; record it as High-severity debt with a deadline, since K1 support is frozen.

## Gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| KSP/Kotlin version mismatch | `ksp-X is too old for kotlin-Y` or cryptic `language version` errors | KSP version string must start with the exact Kotlin version |
| Compose without the new plugin on 2.0+ | `Compose Compiler plugin is required when compose is enabled` | Apply `org.jetbrains.kotlin.plugin.compose` (step 5) |
| Old `kotlinCompilerExtensionVersion` left behind | Ignored silently or sync error depending on AGP | Delete the `composeOptions` version line |
| Hilt + KSP, missed `androidx.hilt` compiler | `@HiltViewModel` classes not injected; runtime crash | Both `hilt-compiler` artifacts move to `ksp(...)` |
| Room schema export silently off after KSP move | Instrumented migration tests fail: no schema JSON | Re-add via `ksp { arg("room.schemaLocation", ...) }` |
| Glide artifact rename missed | kapt removed but no generated GlideApp | Use `com.github.bumptech.glide:ksp`, not `:compiler` |
| Smart-cast compile errors en masse after 2.0 | `Smart cast to 'X' is impossible` across codebase | Local-variable capture pattern; fix while still on 1.9 via the K2 probe |
| `Module was compiled with an incompatible version of Kotlin` | Build fails naming a library | Bump that library to a Kotlin-2.x-built release (often coroutines/serialization) |
| Synthetics imports remain | `Unresolved reference: kotlinx.android.synthetic` | Complete step 6; there is no compatibility shim |
| kapt under K2 fallback | Build warns `Kapt currently doesn't support language version 2.0+. Falling back to 1.9` | Expected if kapt remains; finish KSP migration to remove |
| Parcelize import path | `Unresolved reference: Parcelize` | Plugin `kotlin-parcelize`, import `kotlinx.parcelize.Parcelize` |
| jvmTarget drift in new modules | `Inconsistent JVM-target compatibility` | Keep the alignment from `agp-migration.md` step 4 in every module |

## Documentation Updates

Update `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot (Kotlin old → new, K2 active, kapt fully removed y/n, KSP version, Compose compiler plugin adopted, coroutines/serialization versions) and append a `docs/factory/CHANGELOG.md` entry dated 2026-06-10. Then proceed to `02-engineering/migrations/android-sdk-migration.md`.
