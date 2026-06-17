# Build Verification Gate

This is the universal verification protocol run after **any** code-change phase — every migration
in `02-engineering/migrations/*`, every P6 UI implementation batch, every dependency swap, every
manifest edit. No change is "Done" in `docs/factory/PROJECT_STATE.md` until its mandated tier
passes. The protocol is tiered so that cheap checks run constantly and expensive ones run exactly
when the change type warrants them. The golden rule's second clause ("verify after touching",
`02-engineering/README.md`) is implemented here.

## The Three Tiers

**Tiers 1–3 below are THE canonical verification-tier numbering** (registered in
`01-core/CONVENTIONS.md`). There is no Tier 4 or 5 anywhere in the factory: migration guides map
their release-build and runtime checks **into Tier 3**.

### Tier 1 — Fast (compile gate, ~seconds to 2 min)

```bash
./gradlew :app:compileDebugKotlin
# Multi-module: compile everything cheaply
./gradlew compileDebugKotlin compileDebugJavaWithJavac
```

Pass = zero errors. Warnings: capture count and diff against baseline; new deprecation warnings
after a version bump are expected and become follow-up items, new *error-category* warnings
(`-Werror` candidates) are not.

### Tier 2 — Standard (build + static + tests, ~2–15 min)

```bash
./gradlew assembleDebug
./gradlew :app:lintDebug        # report: app/build/reports/lint-results-debug.html
./gradlew testDebugUnitTest     # report: app/build/reports/tests/testDebugUnitTest/index.html
```

Pass = APK produced, **no new lint errors** vs baseline (lint warnings triaged, not blocking), all
previously-green unit tests still green. If the project has no unit tests, record that fact in the
audit (testability score, `02-engineering/02-architecture-mapping.md`) — Tier 2 then degrades to
assemble + lint, and you note the degraded gate in the CHANGELOG entry.

### Tier 3 — Release-grade (~15–60 min)

```bash
./gradlew clean bundleRelease            # AAB: app/build/outputs/bundle/release/app-release.aab
./gradlew assembleRelease                # if APK also shipped (Huawei/RuStore channels)
# R8 mapping must exist and be non-trivial:
#   app/build/outputs/mapping/release/mapping.txt  (size > ~100 KB for a real app)
# Install release build on emulator/device and run smoke flows (procedure below)
# Upgrade-path install test (procedure below, step 7) when user data is at stake
```

Pass = bundle + mapping produced, release build installs and launches, all smoke flows clean
(including the upgrade-path test when mandated), no FATAL in logcat, size/time baselines within
thresholds (below). Release builds catch what debug never will: R8 stripping reflection targets,
missing keep rules, resource shrinking removing dynamically-referenced drawables, BuildConfig
flag differences.

## Which Tier Is Mandatory When

| Change type | Minimum tier |
|---|---|
| Comment/docs/string resource edits | 1 |
| Logic change inside one module, no dependency/manifest change | 2 |
| Any dependency version bump or addition/removal | 2; **3 if** the dependency ships native libs, uses reflection/annotation processing, or is an ads/billing SDK |
| Gradle / AGP / Kotlin / compileSdk-targetSdk migration (`migrations/*`) | 3 — always |
| Manifest or ProGuard/R8 rule change | 3 |
| Room/SQLite schema, DataStore/prefs, or serialized-model change | 3, **including the upgrade-path install test** (smoke step 7; method in `02-engineering/11-data-migration-safety.md`) |
| DI graph change (Hilt modules, scopes) | 2 + run the app (Hilt fails at runtime graph build) |
| UI implementation batch (P6) | 2 per batch; 3 at batch-set completion |
| Anything in the release branch during P8 | 3 |

One migration at a time: Tier pass → update `PROJECT_STATE.md` + `CHANGELOG.md` → commit → next
migration. Never stack unverified changes.

## Reading Failures — Common Signatures

| Error signature | Likely cause | Fix |
|---|---|---|
| `Unsupported class file major version 6x` | JDK newer than Gradle supports | Align JDK per `08-knowledge/android/version-matrix.md`; set `org.gradle.java.home` or toolchain |
| `Minimum supported Gradle version is X` | AGP bumped past wrapper | Upgrade wrapper first (`02-engineering/migrations/gradle-migration.md` order rule) |
| `Duplicate class ... found in modules` | Conflicting transitives (e.g., listenablefuture, kotlin-stdlib jdk7/8 vs merged) | `dependencyInsight`, then exclude/constraint per `02-engineering/03-dependency-analysis.md` Step 5 |
| `Manifest merger failed : android:exported needs to be explicitly specified` | targetSdk ≥ 31 with old library/manifest | Add explicit `exported` (see `02-engineering/06-manifest-audit.md` Step 2) |
| `Execution failed for task ':app:kaptDebugKotlin'` with obscure `NonExistentClass` | Kapt + new Kotlin incompatibility, or annotation processor needs update | Migrate processor to KSP (`migrations/kotlin-migration.md`); short-term: align processor version |
| `e: ... language version 2.x` / K2 frontend errors in previously-fine code | K2 stricter inference | Fix code (usually real latent bugs); temporary `-language-version 1.9` only as documented stopgap |
| `AAPT: error: resource ... not found` after resource shrink | Shrinker removed dynamically-referenced resource | Add `tools:keep` in `res/raw/keep.xml` |
| Runtime `ClassNotFoundException`/`NoSuchMethodError` only in release | R8 stripped reflection target | Add `-keep` rule; verify against `mapping.txt` |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` on device | Signature mismatch vs installed build | `adb uninstall <applicationId>` then reinstall |
| `Namespace not specified` | AGP 8 requires `namespace` in build file | Move package from manifest to `android { namespace = }` |
| `Cannot use @TaskAction ... was loaded with an old Gradle API` from a plugin | Third-party Gradle plugin too old for new Gradle | Update plugin before Gradle (`migrations/gradle-migration.md` pre-flight) |
| OOM: `Java heap space` during R8/Kapt | Default JVM args too small post-upgrade | `org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g` in `gradle.properties` |
| `ksp-X.Y.Z is too old for kotlin-X.Y.Z` (or KSP plugin not found for the Kotlin version) | KSP version not aligned with Kotlin — KSP versions are `<kotlinVersion>-<kspRelease>` and must match exactly | Bump the KSP plugin to the variant matching the new Kotlin version in the same commit as the Kotlin bump (`migrations/kotlin-migration.md`) |
| `This version (x.y.z) of the Compose Compiler requires Kotlin version a.b.c` | Pre-Kotlin-2.0 standalone Compose compiler pinned to a different Kotlin | On Kotlin 2.0+: apply `org.jetbrains.kotlin.plugin.compose` (version = Kotlin version) and delete `composeOptions`; pre-2.0 stopgap: pin the matching compiler version per `08-knowledge/android/version-matrix.md` |
| Play Console / `zipalign -c -P 16` reports native libs not 16 KB aligned | Old NDK or prebuilt `.so` files built without 16 KB page-size support | Update AGP 8.5.1+/NDK r27+, update or replace the offending SDK's native libs; verify with `zipalign -c -P 16 -v 4 app-release.apk` |
| `Inconsistent JVM-target compatibility ... between 'compileJava' and 'compileKotlin'` | Java `compileOptions` and `kotlinOptions.jvmTarget` diverge after a toolchain bump | Set both via a single Gradle toolchain (`jvmToolchain(17)`); never pin the two separately |

When a failure isn't in this table: re-run with `--stacktrace`, isolate with
`./gradlew :module:taskName`, and check whether the failure reproduces on the pre-change commit
(`git stash` → rerun) before blaming the change.

## Baseline Capture Rule

Before the **first** change of any phase, and after **every** Tier 3 pass, record in the target
repo's `docs/factory/PROJECT_STATE.md` → **Tech Stack Snapshot** (the canonical section name per
`01-core/CONVENTIONS.md` — never "Tech Snapshot" or "Toolchain Versions"):

| Metric | How to measure |
|---|---|
| Clean build time | `./gradlew clean assembleDebug --profile` → `build/reports/profile/`; record total |
| Incremental build time | touch one Kotlin file, re-run `assembleDebug`, record total |
| AAB size | `ls -l app/build/outputs/bundle/release/app-release.aab` (PowerShell: `(Get-Item app\build\outputs\bundle\release\app-release.aab).Length`) |
| Release APK size (universal) | from `assembleRelease` output, or `bundletool build-apks --mode=universal` for true install size |
| Method/dex + resource breakdown (on regressions) | Android Studio Build > Analyze APK…, or `aapt2 dump badging` + APK Analyzer diff |

**Size regression threshold: >5 % growth in AAB size vs the recorded baseline = stop and
investigate** before marking the phase Done (usual culprits: a new SDK pulling transitives, lost
resource shrinking, packaged debug symbols, duplicated native ABIs). Justified growth (new feature
assets) is recorded with its reason in `docs/factory/CHANGELOG.md`; unexplained growth is a
blocker. Build-time regression >20 % likewise gets a profile comparison before sign-off.

## Emulator Smoke Procedure (Tier 3)

```bash
# 1. Boot a target-API emulator (create once via Android Studio Device Manager or avdmanager)
emulator -avd Pixel_8_API_36 -no-snapshot -no-boot-anim &
adb wait-for-device

# 2. Install the RELEASE build (signed with release or upload key)
adb install -r app/build/outputs/apk/release/app-release.apk
# AAB path: java -jar bundletool.jar build-apks --bundle=app-release.aab --output=app.apks --mode=universal
#           java -jar bundletool.jar install-apks --apks=app.apks

# 3. Clear logs, launch, watch for instant crash
adb logcat -c
adb shell monkey -p <applicationId> -c android.intent.category.LAUNCHER 1
sleep 5

# 4. Monkey-lite navigation: 200 pseudo-random events, no system keys, throttled
adb shell monkey -p <applicationId> --throttle 300 --pct-syskeys 0 --pct-anyevent 0 -v 200

# 5. Crash grep
adb logcat -d *:E | grep -E "FATAL|AndroidRuntime"            # bash
adb logcat -d *:E | Select-String "FATAL"                      # PowerShell
adb logcat -d | grep -i "MissingForegroundServiceType\|StrictMode policy violation" # targeted checks

# 6. Manual smoke flows (minimum set — extend per app playbook in 06-playbooks/):
#    cold start → main screen renders; one core user flow end-to-end (per the app's playbook);
#    rotate on main screen (state survives); background + resume; if ads: one ad loads;
#    if billing: purchase sheet opens (test track); deep link: adb shell am start -a android.intent.action.VIEW -d "<link>"

# 7. Upgrade-path install test — MANDATORY when the change touches DB schema, DataStore/prefs,
#    or serialized models; recommended for every release candidate
#    (full method: 02-engineering/11-data-migration-safety.md):
adb uninstall <applicationId>
adb install previous-release.apk             # the last SHIPPED release APK (from release/vX.Y.Z tag or Play)
#    launch and create user data (exercise flows that write DB/prefs)
adb install -r app/build/outputs/apk/release/app-release.apk   # upgrade in place — never uninstall first
#    relaunch and smoke: user data intact, no migration crash
adb logcat -d | grep -iE "IllegalStateException.*migration|SQLiteException"   # bash
adb logcat -d | Select-String -Pattern "migration|SQLiteException"            # powershell
```

Pass = zero FATAL lines, zero ANRs (`adb logcat -d | grep "ANR in"`), all smoke flows completed,
and — when step 7 applies — the upgraded install retains user data and launches clean.
Any crash: pull the stack, fix or revert, re-run full Tier 3 — never ship around a smoke failure.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Tier pass/fail record (tier, date, commit) | `docs/factory/CHANGELOG.md` entry for the change | Phase Done criteria in `01-core/PROJECT_LIFECYCLE.md`; `07-checklists/engineering-modernization.md` |
| Build time + AAB/APK size measurements | `PROJECT_STATE.md` Tech Stack Snapshot (before/after columns) | Size-regression gating; release report (`09-templates/release-report.md`) |
| Lint and test reports (paths) | linked from CHANGELOG entry | P8 `15-final-verification.md` evidence pack |
| New failure signatures encountered + fixes | appended to project notes; recurring ones proposed for `08-knowledge/android/common-pitfalls.md` | Factory knowledge improvement (`10-releases/FRAMEWORK_CHANGELOG.md`) |
| Smoke-flow results (incl. deep links, ads, billing) | CHANGELOG entry | `07-checklists/release-readiness.md`; `16-release-preparation.md` |

If any tier fails and cannot be fixed within the change's effort budget, set the roadmap item to
Blocked in `PROJECT_STATE.md` with the exact failure signature, revert the working tree to the
last green commit, and surface the blocker — a red build is never left overnight as the repo state.
