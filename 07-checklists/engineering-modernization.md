# Engineering Modernization Checklist

Gate for **P4 (Modernization) exit**. This checklist verifies that the target app's build system,
code health, and SDK targets meet the factory tech baseline (June 2026: compileSdk/targetSdk 36,
JDK 17, Kotlin 2.1+, AGP 8.9+, Gradle 8.13+, KSP, version catalogs — see
`08-knowledge/android/version-matrix.md`), that the modernized build is verified per
`02-engineering/07-build-verification.md`, and that project documentation reflects the new state.
P5 (Design Audit & Handover) must not begin until this checklist passes.

> **Run when:** End of P4, after all planned `02-engineering/migrations/*` workstreams are Done or
> Deferred.
> **Run by:** The engineering agent that executed P4 (self-gate), or a fresh agent for audit.
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification.
> Record the completed copy as `docs/factory/reports/engineering-modernization-YYYY-MM-DD.md`.

All commands run from the target app repo root unless noted.

## Build System

- [ ] Gradle wrapper version ≥ 8.13 (or current baseline in
      `08-knowledge/android/version-matrix.md`). Verify: `./gradlew --version` and inspect
      `gradle/wrapper/gradle-wrapper.properties` `distributionUrl`.
- [ ] AGP version is current stable per the baseline (≥ 8.9). Verify: grep `agp` in
      `gradle/libs.versions.toml`; cross-check latest stable before judging.
- [ ] Kotlin version ≥ 2.1 with K2 compiler active. Verify: grep `kotlin` in
      `gradle/libs.versions.toml`; confirm no `kotlin.experimental.tryK2=false` or
      `-language-version 1.9` overrides in `gradle.properties` or build scripts.
- [ ] JDK 17 toolchain declared. Verify: grep `jvmToolchain(17)` or
      `JavaVersion.VERSION_17` in `build.gradle(.kts)` files.
- [ ] Gradle version catalog in use for all dependencies. Verify:
      `gradle/libs.versions.toml` exists and
      `rg "implementation[ (]['\"]" --glob "*.gradle*"` returns no hardcoded
      `group:artifact:version` coordinates (string-literal deps). The `[ (]` covers both
      Kotlin-DSL `implementation("…")` and Groovy `implementation '…'` syntax.
- [ ] Jetifier disabled — no AndroidX migration shim left enabled. Verify:
      `rg "android.enableJetifier" gradle.properties` returns nothing, or the value is `false`
      (a `true` value is a FAIL; legacy support libraries must be removed, not bridged).
- [ ] No `jcenter()` repository anywhere. Verify:
      `rg "jcenter" --glob "*.gradle*" --glob "settings.gradle*"` returns nothing.
- [ ] No dynamic `+` versions. Verify: `rg "[:'\"]\+['\"]" gradle/libs.versions.toml` and
      `rg ":\+" --glob "*.gradle*"` return nothing.
- [ ] No kapt remaining — all annotation processing on KSP. Verify:
      `rg "kapt" --glob "*.gradle*" --glob "*.toml"` returns nothing (comments excepted).

## Code Health

- [ ] No Critical-severity deprecated-API usages remaining (AsyncTask, Loader,
      `PreferenceManager.getDefaultSharedPreferences` for new code, `onBackPressed()` overrides,
      RxJava in new code) — or each remaining usage has a ticket ID in the roadmap with Status and
      a target phase. Verify: `rg "AsyncTask|android.app.LoaderManager|onBackPressed\(\)" --type kotlin --type java`
      and cross-check hits against `docs/factory/audits/ENGINEERING_AUDIT.md` ticket table.
- [ ] All annotation processors (Room, Hilt, Glide, Moshi, etc.) run via KSP. Verify:
      `rg "ksp\(" --glob "*.gradle*"` lists each processor; no `annotationProcessor` entries
      remain (`rg "annotationProcessor" --glob "*.gradle*"`).
- [ ] Lint passes clean or against a committed baseline, with the warning count recorded in this
      report. Verify: `./gradlew :app:lintRelease` exits 0; if `lint-baseline.xml` is used,
      record `rg -c "<issue" app/lint-baseline.xml` here: count = ____.
- [ ] No new compiler warnings introduced versus the pre-P4 build (compare counts from the P3
      audit report `docs/factory/audits/ENGINEERING_AUDIT.md`). Verify:
      `./gradlew :app:compileReleaseKotlin 2>&1 | rg -c "warning:"` — record before/after.

## SDK Targets

- [ ] `compileSdk = 36` and `targetSdk = 36`, or a documented exception exists in
      `docs/factory/PROJECT_STATE.md` (Decision Log section) with reason and revisit date. Verify:
      grep `compileSdk` / `targetSdk` in `app/build.gradle(.kts)` or `libs.versions.toml`.
- [ ] `minSdk` is 26 (or 24 with a recorded legacy-reach justification in PROJECT_STATE). Verify:
      grep `minSdk`.
- [ ] Edge-to-edge handled (mandatory when targeting SDK 35+). Verify: app uses
      `enableEdgeToEdge()` or equivalent inset handling; smoke test shows no content under system
      bars. Check: `rg "enableEdgeToEdge" --type kotlin`.
- [ ] Merged manifest re-audited after P4 dependency changes — no permissions, components, or
      `uses-feature` entries silently introduced or orphaned by migrations. Verify per
      `06-manifest-audit.md` Step 0: build the merged manifest with
      `./gradlew :app:processReleaseMainManifest`, read it at the path that doc's Step 0 states,
      and diff `uses-permission` / `uses-feature` entries against the pre-P4 audit.
- [ ] 16 KB page-size check passed for all native libraries (or N/A if no `.so` files —
      justify with `Get-ChildItem -Recurse -Filter *.so` returning nothing). Verify per
      `02-engineering/migrations/android-sdk-migration.md`; quick check on each `.so` in the
      built APK: `llvm-readelf -l <lib.so> | rg "LOAD"` — alignment must be 0x4000, or use
      Android Studio's APK Analyzer alignment check.

## Release Build Shrinking

- [ ] `minifyEnabled true` **and** `shrinkResources true` on the release build type. Verify:
      `rg "minifyEnabled|shrinkResources" --glob "*.gradle*"` shows both `true` under the release
      build type; cross-check by confirming `app/build/outputs/mapping/release/mapping.txt` is
      produced by `./gradlew :app:assembleRelease`.
- [ ] R8 keep-rules reviewed after enabling full mode — rules are minimal and justified, not a
      blanket `-keep class ** { *; }`. Verify per `02-engineering/09-r8-shrinking.md`: read
      `proguard-rules.pro`, confirm each `-keep` has a stated reason (reflection, serialization,
      JNI) and no catch-all keep masks shrinking. Record the rule count.

## Behavioral Safety Net

- [ ] Characterization / test safety net exists for the surfaces touched in P4, per
      `02-engineering/08-testing-strategy.md` — enough coverage to catch a regression in the
      migrated paths. Verify: the strategy doc's required tests exist and run
      (`./gradlew :app:testReleaseUnitTest` lists them); record the covered surfaces.
- [ ] Process-death / state-restoration test run on a P4-touched flow: kill the process mid-flow
      and confirm the app restores state without crashing. Verify: open a key flow, then
      `adb shell am kill <package>` (or background + `adb shell am kill`), relaunch, confirm no
      crash and state restored; `adb logcat -d *:E | rg "FATAL"` empty for the session.

## Verification

- [ ] Tier 3 build verification per `02-engineering/07-build-verification.md` is green: clean
      build, unit tests, lint, release artifact assembly. Verify:
      `./gradlew clean :app:testReleaseUnitTest :app:lintRelease :app:bundleRelease` exits 0;
      attach the final BUILD SUCCESSFUL line.
- [ ] APK/AAB size delta versus pre-P4 baseline recorded and < 5%, or the increase is justified
      in writing here. Verify: compare `app/build/outputs/bundle/release/app-release.aab` byte
      size against the baseline recorded in `docs/factory/audits/ENGINEERING_AUDIT.md`.
      Before = ____ bytes, After = ____ bytes, Delta = ____%.
- [ ] Smoke test passed on a device/emulator running the highest available API level: install,
      cold start, navigate the top-3 user flows, no crash. Verify:
      `adb install -r app/build/outputs/apk/release/app-release.apk` then manual/scripted pass;
      `adb logcat -d *:E | rg "FATAL"` returns nothing for the session.

## Documentation

- [ ] `docs/factory/PROJECT_STATE.md` Tech Stack Snapshot updated with the post-P4 versions
      (Gradle, AGP, Kotlin, compileSdk/targetSdk/minSdk, key libraries). Verify: open the file;
      values match the verified versions above.
- [ ] Every P4 roadmap workstream in `docs/factory/PROJECT_STATE.md` has Status = Done or
      Deferred (with reason). Verify: no workstream remains Not Started / In Progress / Blocked.
- [ ] `docs/factory/CHANGELOG.md` contains dated entries for each migration performed (Gradle,
      AGP, Kotlin, SDK, kapt→KSP, etc.). Verify: read the changelog; each Build System item above
      that changed has a corresponding entry.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / FAIL |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |

On PASS: advance the phase tracker in `docs/factory/PROJECT_STATE.md` to P5 and log this report in
`docs/factory/CHANGELOG.md`. On FAIL: file remediation actions, fix, and re-run failed groups in a
new dated report.
