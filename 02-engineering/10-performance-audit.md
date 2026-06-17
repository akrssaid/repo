# Performance Audit — Startup, Jank, Memory, Size, Vitals Triage

This method produces the performance evidence for audit dimension 7 of
`docs/factory/audits/ENGINEERING_AUDIT.md` (prompt 03), the before/after numbers around P4/P6
work, and the Vitals triage that `10-releases/release-workflow.md` stage 5 executes during
rollout. **All numeric thresholds live in `08-knowledge/android/play-vitals-performance.md`**
(startup-time bands, crash/ANR bad-behavior lines); this doc defines *how to measure and triage*,
never the numbers. Findings are recorded as `ENG-NNN` rows; scales per `01-core/CONVENTIONS.md`.

## 1. App Startup

Definitions (Android Vitals semantics):

- **Cold start** — process not alive; includes process fork, Application init, first frame.
  The number that matters most; the one with bands in `play-vitals-performance.md`.
- **Warm start** — process alive, activity recreated (e.g., evicted activity, back navigation
  into the task). Mostly activity `onCreate` → first frame.
- **Hot start** — activity resumed from background; should be near-instant.

**Measurement** — `am start -W` prints `TotalTime` (app's own startup: process start → first
frame drawn) and `WaitTime` (TotalTime + system overhead handing off). Read **TotalTime** for
app-attributable startup; a large WaitTime−TotalTime gap is device/system noise, not your bug.
Always average ≥5 runs and force-stop between runs to keep them cold:

```bash
PKG=<applicationId>; ACT=$(adb shell cmd package resolve-activity --brief $PKG | tail -1)
for i in 1 2 3 4 5; do
  adb shell am force-stop $PKG
  sleep 2
  adb shell am start -W -n $ACT | grep -E "TotalTime|WaitTime"
done
```

```powershell
$pkg = "<applicationId>"
$act = (adb shell cmd package resolve-activity --brief $pkg | Select-Object -Last 1)
1..5 | ForEach-Object {
  adb shell am force-stop $pkg
  Start-Sleep -Seconds 2
  adb shell am start -W -n $act | Select-String "TotalTime|WaitTime"
}
```

For warm starts: skip the `force-stop`, press back (`adb shell input keyevent 4`) before
relaunching. Record min/median/max of TotalTime in the audit; measure on a release (R8) build —
debug builds are 30–50% slower and not the shipped reality.

**TTID vs TTFD**: `TotalTime` is time-to-initial-display. If the first frame is a skeleton and
real content arrives later, call `Activity.reportFullyDrawn()` when content is actually usable —
logcat then prints `Fully drawn <pkg>: +NNNms`, which is the honest TTFD number and what Play
attributes when present:

```bash
adb logcat -d | grep "Fully drawn"
```

```powershell
adb logcat -d | Select-String "Fully drawn"
```

Verdict bands (good/acceptable/investigate) per cold/warm/hot:
`08-knowledge/android/play-vitals-performance.md` — cite it, never restate the seconds here.

## 2. Baseline Profiles

A Baseline Profile is an AOT-compilation hint shipped inside the AAB: the listed classes/methods
are compiled at install time instead of after JIT warm-up, typically cutting cold start 15–30%
and removing first-scroll jank. **When they pay**: startup-heavy apps, list-scroll-jank apps,
anything where the audit's §1 numbers sit in the "acceptable" band and the roadmap wants "good".
**Cost**: a `macrobenchmark` module and a managed-device generation task — size the work M.

Two routes:

1. **Library-provided profiles** — Compose, and most Jetpack libraries, already ship their own
   baseline profiles; merely being on current versions (P4) captures most of the win. Verify
   before building custom infrastructure.
2. **App-specific profile** via a macrobenchmark module (minimal sketch):

```kotlin
// :baselineprofile module, src/main/java/.../BaselineProfileGenerator.kt
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test fun generate() = rule.collect(packageName = "<applicationId>") {
        pressHome(); startActivityAndWait()        // cold start path
        // + scroll the main list / run the P0 flow — journeys to pre-compile
    }
}
```

Wire with the `androidx.baselineprofile` Gradle plugin (target `:app`, producer
`:baselineprofile`); generate on a managed device or rooted emulator:

```bash
./gradlew :app:generateBaselineProfile
```

The profile lands in `app/src/release/generated/baselineProfiles/` and ships automatically in
`bundleRelease`. **Verification on device** (after installing via Play internal track or
`bundletool` local-testing — profiles are install-time compiled):

```bash
adb shell dumpsys package dexopt | grep -A 2 "<applicationId>"
# want: status=speed-profile, and "[reason=install" or baseline-related compile reason
```

```powershell
adb shell dumpsys package dexopt | Select-String -Context 0,2 "<applicationId>"
```

Then re-run §1's loop and record the delta in the audit/CHANGELOG.

## 3. Jank (Rendering)

```bash
adb shell dumpsys gfxinfo <applicationId> reset    # zero counters
# ... exercise the janky flow for ~30 s ...
adb shell dumpsys gfxinfo <applicationId>
```

(Same commands in PowerShell.) Read the summary block: **Total frames rendered**, **Janky
frames** (count + %), and the **percentile rows** (90th/95th/99th frame times). Triage:

| Reading | Meaning | First move |
|---|---|---|
| Janky % high, 99th-pctile huge, 90th fine | Occasional hitches (GC, lazy init on touch, image decode on main) | Logcat around the hitch; StrictMode in debug |
| 90th pctile already > frame budget | Systematic overdraw / heavy layouts / recomposition storm | Layout Inspector; flatten hierarchy or fix Compose keys |
| `Number Missed Vsync`, high-input-latency rows climbing | Main thread blocked in input handling | Method trace the touch handler |

**Compose recomposition**: Android Studio Layout Inspector → Recomposition counts (counts/skips
per composable) — a list row recomposing on every scroll frame is the canonical finding. For
release-build evidence, enable composition tracing (`androidx.compose.runtime:runtime-tracing`)
and capture a system trace; record the offending composable in the ENG finding. Common fixes:
unstable lambda/param (hoist or `remember`), missing `key()` in lazy lists, reading a frequently
changing `State` too high in the tree.

## 4. Memory

- **LeakCanary in debug is the factory standard**: `debugImplementation
  "com.squareup.leakcanary:leakcanary-android:<version>"` — zero code. Any retained-object
  report on a P0 flow is an ENG finding (Severity High if it leaks an Activity/Context).
- **Triage snapshot**:

```bash
adb shell dumpsys meminfo <applicationId>
```

  Read **TOTAL PSS** (overall footprint), **Activities/Views counts** in the Objects section
  (Activities > number of screens on the back stack = leak; Views in the thousands on a simple
  screen = inflation leak), and **Native Heap** (bitmap/JNI growth). Capture before/after a
  suspect flow plus `am broadcast`-free idle; growth that never returns after backgrounding +
  `adb shell am send-trim-memory <pkg> RUNNING_CRITICAL` is the smoking gun.
- **Heap dump** when LeakCanary can't run (release build / instrumented repro):

```bash
adb shell am dumpheap <applicationId> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof
# open in Android Studio Profiler (auto-converts) or: hprof-conv heap.hprof heap-mat.hprof
```

```powershell
adb shell am dumpheap <applicationId> /data/local/tmp/heap.hprof
adb pull /data/local/tmp/heap.hprof .
```

## 5. APK/AAB Size

Budget rule and baseline capture live in `02-engineering/07-build-verification.md` (record AAB
size every Tier 3; **>5% unexplained growth = blocker**). The diff procedure when the budget
trips:

```bash
# apkanalyzer ships in cmdline-tools
apkanalyzer apk file-size app-release.apk
apkanalyzer apk download-size app-release.apk          # what users actually pay
apkanalyzer apk compare --different-only old.apk new.apk   # per-path delta — the actual diff
apkanalyzer dex packages --defined-only app-release.apk | sort -k2 -n -r | head -25
```

```powershell
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\apkanalyzer.bat" apk compare --different-only old.apk new.apk
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\apkanalyzer.bat" dex packages --defined-only app-release.apk | Sort-Object { [int]($_ -split '\s+')[1] } -Descending | Select-Object -First 25
```

For AABs, compare the universal APKs built by `bundletool build-apks --mode=universal`, or use
Android Studio → Build → Analyze APK… → Compare. Usual culprits and owners: new SDK transitives
(`03-dependency-analysis.md`), lost resource/code shrinking (`09-r8-shrinking.md`), duplicated
native ABIs, unstripped debug symbols.

## 6. Play Vitals Triage (release-workflow stage 5 executes this)

During rollout, read Play Console → Quality → Android Vitals. Thresholds and HOLD lines:
`08-knowledge/android/play-vitals-performance.md` + `10-releases/release-workflow.md` stage 4.
Method: sort crash and ANR **clusters** by affected-user count (never raw event count — one user
in a crash loop inflates events); deobfuscate per `09-r8-shrinking.md` §7; classify each cluster
by failure class and route it to its owner:

| Failure class (cluster signature) | Likely origin | Routes back to |
|---|---|---|
| `ClassNotFoundException`/`NoSuchMethodError`/null-field JSON, release-only | Shrinking | `02-engineering/09-r8-shrinking.md` §9; fix → P8 fast path |
| Crash in `Migration`/`SQLiteException`, spikes on update-installs only | DB migration | `02-engineering/11-data-migration-safety.md`; **forward-fix only** |
| ANR cluster in `Application.onCreate`/ContentProvider init | Startup work on main thread | This doc §1–2; roadmap item (P4) |
| ANR in broadcast/service, new after targetSdk hop | SDK behavior change | `migrations/android-sdk-migration.md` behavior checklist |
| Crash inside an ads/billing SDK version | SDK regression | `02-engineering/03-dependency-analysis.md` (pin/rollback dep); escalate per SDK vendor |
| OOM clusters, bitmap/native traces | Memory (§4) | ENG finding → P4 backlog; HOLD only if growing |
| Crash in a redesigned screen's Compose stack | P6 implementation | Prompt 09 ticket reopened; `07-checklists/design-modernization.md` |
| Startup-time vitals band regression, no crash | Perf regression | This doc §1/§2; compare against PROJECT_STATE baselines |

Every triaged cluster gets a row in the release report (`09-templates/release-report.md`) with:
cluster signature, affected users %, failure class, route, decision (fix-forward / hold / watch).
HOLD/halt decisions themselves belong to release-workflow stage 4's decision table — this method
only supplies the classified evidence.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Startup table (cold/warm/hot, 5-run min/median/max, TTFD) | `audits/ENGINEERING_AUDIT.md` dimension 7; PROJECT_STATE Tech Stack Snapshot | Roadmap (P4 perf items); release report before/after |
| Baseline Profile delta | CHANGELOG entry | Release report; `play-vitals-performance.md` band check |
| Jank/memory findings (`ENG-NNN`) | `audits/ENGINEERING_AUDIT.md` | Prompt 04 backlog; P6 ticket constraints |
| Size diff analysis | CHANGELOG entry | `07-build-verification.md` size gate |
| Vitals triage table per rollout stage | `reports/RELEASE_REPORT_v<versionName>.md` | Release-workflow stage 4 decisions; stage 7 retro |
