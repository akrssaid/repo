# Android SDK Migration — Raising compileSdk / targetSdk

Execution-grade guide for raising `compileSdk` and `targetSdk` during lifecycle phase **P4 — Modernization**. An agent follows this step-by-step on a real codebase. Unlike the toolchain layers before it, a `targetSdk` bump changes **runtime behavior on user devices** — every bump has a per-version behavior-change checklist that must be tested, not just compiled.

## Mandatory Global Ordering

```text
Gradle → AGP → Kotlin → compileSdk/targetSdk → libraries
```

**Why:** each layer's minimum requirements sit on the layer below. New SDK levels require minimum AGP versions to compile (AGP must already be modern — see `02-engineering/migrations/agp-migration.md`), and AGP requires its Gradle (see `gradle-migration.md`); AndroidX libraries you'll bump afterwards require minimum compileSdk levels (so SDK precedes libraries). This guide assumes Gradle, AGP, and Kotlin migrations (`kotlin-migration.md`) are complete and green.

## Preconditions

| Requirement | Check | If missing |
|---|---|---|
| Toolchain layers done | Gradle/AGP/Kotlin at audit targets, baseline green | Complete prior guides |
| Pre-migration test baseline | Test coverage meets the characterization minimum for an SDK-level migration per the migration-type table in `02-engineering/08-testing-strategy.md` | Write the mandated characterization tests **before** bumping — do not assume previously-green tests exist or suffice |
| Current SDK levels known | `compileSdk`, `targetSdk`, `minSdk` from module build files / `gradle/libs.versions.toml` | Record in `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot |
| Target level chosen | Per `docs/factory/audits/ENGINEERING_AUDIT.md`; factory baseline compileSdk = targetSdk = 36 | See Play deadline rule below |
| Manifest audit done | `02-engineering/06-manifest-audit.md` output available (permissions, services, exported components) | Run it — the behavior checklists key off it |
| Test devices/emulators | At least one device or AVD **running each Android version you are newly targeting** | Create AVDs (API 33/34/35/36 as applicable); behavior changes only manifest on those OS versions |
| Native libs inventoried | `unzip -l app-release.apk | grep "\.so$"` or APK Analyzer | Needed for the 16 KB page-size check (step 6) |

**Play Console target-API deadline rule:** new apps and app updates must target an API level no more than one year older than the latest stable Android release; the enforcement window cycles every August (as of 2026-06-10, that means targetSdk 35 minimum now, 36 required from August 2026). **Verify the current deadline at support.google.com/googleplay/android-developer (target API level requirements) before planning** — missing it blocks releases, which is a P8 hard failure.

## Pre-flight

```bash
git tag pre-sdk-migration-2026-06-10
git checkout -b chore/sdk-bump
./gradlew clean assembleDebug testDebugUnitTest --stacktrace 2>&1 | tee docs/factory/reports/baseline-sdk-build.log
```

Capture the current merged manifest for diffing: obtain it per `02-engineering/06-manifest-audit.md` Step 0 (it owns the build task and output path for both AGP-version variants), then copy it to `docs/factory/reports/baseline-merged-manifest.xml`.

## Procedure

### 1. Bump compileSdk first — compile-time only, safe to do in one jump

`compileSdk` changes what APIs you compile against; it does **not** change runtime behavior. Bump it straight to the final target:

```diff
 # gradle/libs.versions.toml
 [versions]
-compileSdk = "33"
+compileSdk = "36"
```

```diff
 // app/build.gradle.kts
 android {
-    compileSdk = 33
+    compileSdk = 36
```

Confirm your AGP can compile the target level (ordering guard — if this fails you skipped a layer):

| compileSdk | Minimum AGP |
|---|---|
| 34 | 8.1.1 |
| 35 | 8.4 (8.6 recommended) |
| 36 | **8.9.1** |

This table matches `agp-migration.md` and the canonical version matrix in `08-knowledge/android/version-matrix.md`; verify the current pairing at developer.android.com/build/releases/gradle-plugin. Install the platform if needed:

```bash
"$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager" "platforms;android-36"
```

```powershell
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\sdkmanager.bat" "platforms;android-36"
```

Build and fix compile errors and **new deprecation warnings** (deprecated APIs are your early warning for the targetSdk work below). Commit: `chore: compileSdk 33 -> 36`.

### 2. Bump targetSdk one level at a time, working each behavior checklist

`targetSdk` opts you into runtime behavior changes. Hop **one level per commit**, and run the relevant checklist on a device running that OS version before the next hop.

```diff
-targetSdk = "33"
+targetSdk = "34"
```

#### Checklist → 33 (Android 13)

| Change | Affects | Action |
|---|---|---|
| Notification runtime permission | Every app posting notifications | Add `<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>`; request at a contextual moment (not cold start): `ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.POST_NOTIFICATIONS), RC)` gated on `Build.VERSION.SDK_INT >= 33`; handle denial gracefully (app must still function) |
| Granular media permissions | Apps reading shared media | Replace `READ_EXTERNAL_STORAGE` with `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` / `READ_MEDIA_AUDIO` (only those needed); keep `READ_EXTERNAL_STORAGE` with `android:maxSdkVersion="32"` |
| Clipboard preview / intent filters block non-matching intents | Apps with exported components | Verify exported components handle only declared intent shapes |
| Predictive back available as opt-in | Apps with custom back handling | Optional early prep for the 36 default-on: set `android:enableOnBackInvokedCallback="true"` and migrate to `OnBackPressedCallback` now |

#### Checklist → 34 (Android 14)

| Change | Affects | Action |
|---|---|---|
| Foreground service types mandatory | Every `startForeground()` user | Declare a type per service in the manifest **and** the matching permission, e.g. `<service android:name=".SyncService" android:foregroundServiceType="dataSync"/>` + `<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC"/>`; pass the type in `ServiceCompat.startForeground(svc, id, notif, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)`. Wrong/missing type → `MissingForegroundServiceTypeException` at runtime |
| Implicit intent restrictions | Senders of implicit intents to internal components | Implicit intents no longer delivered to non-exported components; use explicit `Intent(context, Target::class.java)`; mutable PendingIntents must not wrap implicit intents |
| Partial media access (`READ_MEDIA_VISUAL_USER_SELECTED`) | Media-reading apps | Add the permission; handle the "user selected some photos" grant state in the permission flow UI; re-query selection on each access |
| `BroadcastReceiver` context-registered must declare exported flag | Dynamic receivers | `ContextCompat.registerReceiver(ctx, receiver, filter, ContextCompat.RECEIVER_NOT_EXPORTED)` (or `RECEIVER_EXPORTED` only when external broadcasts expected) |
| Exact alarms denied by default | Alarm/reminder apps using `SCHEDULE_EXACT_ALARM` | Newly installed apps targeting 34 no longer get the permission pre-granted; check `AlarmManager.canScheduleExactAlarms()`, route users to `ACTION_REQUEST_SCHEDULE_EXACT_ALARM`, or use `USE_EXACT_ALARM` only if the app is genuinely an alarm/calendar app (Play policy-gated) |
| Minimum installable SDK floor | Very old minSdk values | Devices refuse installs of apps targeting <23; irrelevant if you follow this guide, but flags abandoned forks |

#### Checklist → 35 (Android 15)

| Change | Affects | Action |
|---|---|---|
| **Edge-to-edge enforced** | Every activity — app draws behind system bars whether ready or not | Remediation below |
| Status/nav bar color APIs deprecated | `Window.setStatusBarColor`, `setNavigationBarColor`, related theme attrs are no-ops/deprecated when targeting 35 | Remove them; bar areas show app content — control appearance via the content you draw behind the bars + `WindowInsetsControllerCompat.isAppearanceLightStatusBars` |
| `elegantTextHeight` / font metric changes | Text-heavy UIs in some scripts | Visual regression pass |

**Edge-to-edge remediation — Compose** (requires `androidx.activity:activity-compose` 1.8+ — bump it here if older; that is a sanctioned exception to "libraries last" because remediation depends on it):

```kotlin
// Activity
override fun onCreate(savedInstanceState: Bundle?) {
    enableEdgeToEdge()   // androidx.activity 1.8+; aligns pre-35 behavior with 35
    super.onCreate(savedInstanceState)
    setContent { AppTheme { AppScaffold() } }
}

// Composables: consume insets via Scaffold or modifiers
Scaffold { innerPadding ->
    Content(Modifier.padding(innerPadding))
}
// Non-Scaffold roots:
Box(Modifier.fillMaxSize().windowInsetsPadding(WindowInsets.safeDrawing)) { ... }
```

**Edge-to-edge remediation — Views:**

```kotlin
WindowCompat.setDecorFitsSystemWindows(window, false)
ViewCompat.setOnApplyWindowInsetsListener(binding.root) { v, insets ->
    val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
    v.updatePadding(top = bars.top, bottom = bars.bottom)
    WindowInsetsCompat.CONSUMED
}
```

Audit every screen on an API-35 device: content under the status bar, FABs/bottom bars under gesture nav, keyboards overlapping inputs (`WindowInsets.ime` / `Type.ime()`). Coordinate with `03-design/05-material3-modernization.md` if a redesign is queued — do not double-pay insets.

**Temporary escape hatch (35 only):** if the release deadline arrives before all screens are remediated, the theme attribute below defers enforcement — but it is deprecated and **ignored once you target 36**, so it only buys one cycle. Record it as High-severity debt with a deadline if used:

```xml
<style name="AppTheme" parent="...">
    <item name="android:windowOptOutEdgeToEdgeEnforcement">true</item>
</style>
```

#### Checklist → 36 (Android 16) — highlights

| Change | Affects | Action |
|---|---|---|
| Predictive back default-on | Apps overriding back (`onBackPressed`, `OnBackPressedCallback` misuse) | Migrate to `OnBackPressedCallback`/`androidx.activity` back APIs; test back-gesture previews on every screen with custom back handling; opt-out (`android:enableOnBackInvokedCallback="false"`) only as recorded debt |
| Adaptive layouts expectations | Apps locking orientation / fixed-size assumptions on large screens | Orientation/resizability restrictions are ignored on large screens for 36-targeting apps; test foldable/tablet AVDs; adopt `WindowSizeClass`-driven layouts |
| 16 KB page size enforcement tightens | Apps with native libs | Step 6 is mandatory |

**Verify the current Android 16 behavior-change list at developer.android.com/about/versions/16/behavior-changes-16 before executing this hop** — 36 is the newest level and its list is the most likely to have grown since this guide was written (2026-06-10).

### 3. Test each hop at runtime — compiling is not done

For each targetSdk hop, on a device/AVD running that OS version, exercise: app cold start, notification posting (33+), every foreground service start path (34+), media pick/read flows (33/34+), every screen's insets (35+), back gesture on screens with custom back logic (36). Additionally run the **upgrade-path install test** from `02-engineering/11-data-migration-safety.md`: install the currently-released build, populate state (DB rows, preferences, granted permissions, logged-in session), then install the bumped build over it with `adb install -r` — targetSdk behavior changes interact with persisted state and prior permission grants on *upgrading* users, which fresh installs never exercise. Watch:

```bash
adb logcat -s AndroidRuntime:E StrictMode:W
```

Use the compatibility framework to preview a behavior change **before** the bump (debuggable builds only):

```bash
adb shell am compat enable FORCE_NON_RESIZE_APP com.example.app   # example change ID
adb shell am compat reset-all com.example.app                     # restore defaults after testing
```

Change IDs for each release are listed on the corresponding developer.android.com behavior-changes page.

### 4. minSdk policy — raise only with operator approval

Never raise `minSdk` as a side effect of this migration. Raising it drops installed users and shrinks reach. Procedure if a library bump (later layer) demands it:

1. Pull install-base distribution from Play Console (Statistics → Android version) for this app.
2. Compute the % of installs below the proposed minSdk.
3. Present to the operator: proposed minSdk, % users dropped, which dependency forces it, alternatives (older library version, desugaring).
4. Proceed only on explicit approval; record the decision in `docs/factory/PROJECT_STATE.md` → Decision Log. Factory recommendation: minSdk 26 (24 acceptable for legacy-reach apps) per `08-knowledge/android/version-matrix.md`.

Before raising minSdk for an API-availability reason, check whether **core library desugaring** solves it instead (gives `java.time`, `java.util.stream` etc. down to old API levels):

```kotlin
android {
    compileOptions { isCoreLibraryDesugaringEnabled = true }
}
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.5")
}
```

### 5. Manifest re-audit

Rebuild the merged manifest per `02-engineering/06-manifest-audit.md` Step 0 (it owns the build task and output path), then diff it against the pre-flight snapshot:

```bash
git diff --no-index docs/factory/reports/baseline-merged-manifest.xml <merged-manifest-path-from-06-manifest-audit-step-0>
```

Confirm: new permissions are intentional (libraries sometimes inject them — strip with `<uses-permission android:name="..." tools:node="remove"/>`), foreground service types present, no unexpected exported components. Cross-check against `02-engineering/06-manifest-audit.md` findings.

### 6. 16 KB page-size native-library compliance

Devices with 16 KB memory pages are shipping; Play requires 16 KB compatibility for apps with native code targeting recent SDKs. Check every `.so`:

```bash
# Quick alignment check on the packaged APK (zipalign from build-tools 35+)
"$ANDROID_HOME/build-tools/36.0.0/zipalign" -c -P 16 -v 4 app/build/outputs/apk/release/app-release.apk
```

```powershell
& "$env:ANDROID_HOME\build-tools\36.0.0\zipalign.exe" -c -P 16 -v 4 app\build\outputs\apk\release\app-release.apk
```

Exit code 0 = compliant. Alternatively, Android Studio APK Analyzer opens the APK and flags non-16-KB-aligned libraries. If you ship an AAB (normal for Play), check the device-served APKs, not the bundle itself:

```bash
java -jar bundletool.jar build-apks --bundle=app/build/outputs/bundle/release/app-release.aab --output=probe.apks --mode=universal
# Extract universal.apk from probe.apks (it is a zip), then run the zipalign -c -P 16 check on it
```

For deeper inspection of a single lib — substitute your installed NDK version for `<version>` (list installed versions: `ls "$ANDROID_HOME/ndk"` / `Get-ChildItem "$env:ANDROID_HOME\ndk"`):

```bash
"$ANDROID_HOME/ndk/<version>/toolchains/llvm/prebuilt/windows-x86_64/bin/llvm-readelf" -l lib/arm64-v8a/libfoo.so | grep LOAD
# Align column must show 0x4000 (16 KB), not 0x1000
# (prebuilt dir is windows-x86_64 on Windows hosts; linux-x86_64 / darwin-x86_64 elsewhere)
```

```powershell
& "$env:ANDROID_HOME\ndk\<version>\toolchains\llvm\prebuilt\windows-x86_64\bin\llvm-readelf.exe" -l lib\arm64-v8a\libfoo.so | Select-String LOAD
# Align column must show 0x4000 (16 KB), not 0x1000
```

Remediation: rebuild your own NDK code with AGP 8.5.1+/NDK r27+ (16 KB alignment is the default there; for older NDK add `-Wl,-z,max-page-size=16384` to ldflags); for third-party `.so` dependencies, upgrade to a release that ships 16 KB-aligned binaries — check the vendor's changelog, and record a Critical blocker in `docs/factory/PROJECT_STATE.md` → Known Risks if none exists. Apps with zero native libs: record "N/A — no native code" and move on.

## Verification

Run the canonical tiers from `02-engineering/07-build-verification.md`. Toolchain migrations always require Tier 3 (see its mandatory-tier table):

| Tier | Command | Pass criterion |
|---|---|---|
| 1 — Fast | `./gradlew help` then `./gradlew compileDebugKotlin compileDebugJavaWithJavac` | No errors against new compileSdk |
| 2 — Standard | `./gradlew clean assembleDebug :app:lintDebug testDebugUnitTest` | BUILD SUCCESSFUL; review new `NewApi` / `UnusedAttribute` lint findings; characterization tests (per `02-engineering/08-testing-strategy.md`) pass |
| 3 — Release-grade | `./gradlew clean bundleRelease assembleRelease` + zipalign 16 KB check (step 6) + step 3 runtime matrix (incl. the upgrade-path install test) on devices at each newly-targeted OS level **plus** one minSdk-level device | Minified build passes; alignment compliant; all behavior checklists pass; no regressions on oldest supported OS |

Tier 3's runtime matrix carries the real risk for this migration — schedule it on physical hardware where possible (notification permission and edge-to-edge differ subtly on OEM skins).

Additionally, before the production rollout (P8), upload the bumped build to a Play **internal testing** track and review the Pre-launch report — it exercises the app on physical devices at current OS levels and reliably catches edge-to-edge clipping and foreground-service crashes that local AVDs miss. Treat Pre-launch report crashes as Tier 3 failures.

## Rollback

```bash
git revert <hop-commit-sha>     # roll back one targetSdk level; checklists are per-hop so this is clean
git checkout main && git branch -D chore/sdk-bump   # full layer
```

`compileSdk` rollback is safe anytime (compile-time only). A `targetSdk` rollback after a Play release is **not** possible below the current Play deadline floor — if a behavior change breaks production, you must fix forward (or roll back to the highest compliant level only). This asymmetry is why Tier 3 runtime testing happens before release, per `07-checklists/release-readiness.md`.

## Gotchas

| Gotcha | Symptom | Fix |
|---|---|---|
| Bumping targetSdk without runtime testing | Compiles clean, breaks on user devices (notifications silent, services crash) | Behavior checklists are runtime work; step 3 is mandatory |
| Notifications silently not shown on 33+ | No crash, no notification | `POST_NOTIFICATIONS` not granted; implement the request flow and check `areNotificationsEnabled()` |
| `MissingForegroundServiceTypeException` on 34+ | Crash at `startForeground()` | Manifest type + permission + matching `startForeground` type constant (all three) |
| Library injects max-sdk-stale permissions | Unexpected `READ_EXTERNAL_STORAGE` in merged manifest | `tools:node="remove"` override; verify in merged manifest |
| Edge-to-edge "fixed" only on home screen | Other screens still draw under bars on 35 | Insets are per-screen; audit every activity/nav destination |
| `setStatusBarColor` no-op confusion | Status bar "wrong color" on 35+ | API is deprecated/ignored; draw the color in content behind the bar instead |
| Double insets after `enableEdgeToEdge` + Scaffold + manual padding | Excess top/bottom padding | Apply insets once: Scaffold's `innerPadding` **or** `windowInsetsPadding`, never both |
| Predictive back skips custom cleanup on 36 | Custom `onBackPressed` logic never runs | Migrate to `OnBackPressedCallback` registered via `onBackPressedDispatcher` |
| Old AGP can't compile new SDK | `compileSdk 36 requires AGP X+` warning/error | Ordering violation — finish `agp-migration.md` first |
| 16 KB check run on debug APK only | Release packaging differs (uncompressed libs) | Run zipalign check on the **release** artifact (and the AAB-derived APKs via `bundletool build-apks`) |
| AVD missing for new API level | "Testing" done on old OS, changes never exercised | `sdkmanager "system-images;android-36;google_apis;x86_64"` + create AVD before declaring Tier 3 done |
| Behavior changes tested only on newest OS | Regressions on minSdk-level devices missed | Tier 3 runtime matrix includes one device at minSdk; `Build.VERSION.SDK_INT` guards must be exercised on both sides |
| compileSdk and targetSdk bumped in one commit | Compile fixes and runtime behavior changes interleaved; bisecting impossible | Step 1 (compileSdk, one commit) strictly before step 2 (targetSdk, one commit per level) |
| targetSdk floor after release | Cannot revert below Play deadline once shipped | Fix forward; never ship a hop whose checklist wasn't device-tested |
| `windowOptOutEdgeToEdgeEnforcement` treated as a fix | Edge-to-edge "solved" on 35, breaks again targeting 36 | It is a deferral, not a fix; ignored at targetSdk 36 — do the insets work |
| Exact alarms silently not firing on 34+ | Reminders/alarms missed, no crash | `canScheduleExactAlarms()` check + request flow; fall back to inexact `setAndAllowWhileIdle` where acceptable |

## Documentation Updates

Update `docs/factory/PROJECT_STATE.md` → Tech Stack Snapshot (compileSdk/targetSdk old → new, minSdk unchanged or approved change with Decision Log link, 16 KB compliance status, per-hop checklist completion) and append a `docs/factory/CHANGELOG.md` entry dated 2026-06-10. P4 toolchain ordering then continues to library upgrades per `02-engineering/03-dependency-analysis.md` and the roadmap from `01-core/prompts/04-roadmap-generation.md`.
