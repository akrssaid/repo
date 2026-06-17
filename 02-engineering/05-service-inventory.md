# App Components & Background Work Inventory Method

This method inventories every runtime surface of the target app — activities, services, receivers,
providers, background-work schedulers, deep links, widgets, and shortcuts — and maps each legacy
pattern to the exact Android API level where it breaks. It runs in P3 under
`01-core/prompts/03-modernization-audit.md`; its risk table is the backbone of the modernization
roadmap's "behavior changes" track, because raising `targetSdk` (mandated by Play every August)
is exactly the act that detonates these patterns. Work from the **merged** manifest, not just
`src/main` — libraries merge in components of their own.

## Step 1 — Component Inventory from the Merged Manifest

Obtain the merged manifest via `02-engineering/06-manifest-audit.md` **Step 0** — it owns the
build command and both AGP-version path variants; never reconstruct the path from memory.

Build one table per component type. Required columns:

| Component | Class | `exported` | Intent filters | Permission guard | Source (app or which library) | Purpose |
|---|---|---|---|---|---|---|

- **Activities**: mark the launcher; note `taskAffinity`/`launchMode` oddities (forward to
  `06-manifest-audit.md`); note alias activities (icon switching).
- **Services**: record `android:foregroundServiceType` (or its absence — see Step 3),
  `android:process` (multi-process apps complicate SDK init), and whether bound, started, or both
  (read the class: `onBind` vs `onStartCommand`).
- **Receivers**: which are manifest-registered vs context-registered — bash:
  `grep -rn --include="*.kt" --include="*.java" "registerReceiver(" app/src`; powershell:
  `Get-ChildItem -Recurse app\src -Include *.kt,*.java | Select-String "registerReceiver\("`.
  Manifest-registered
  implicit-broadcast receivers mostly stopped working at API 26 — verify each one's action still
  delivers. Context-registered receivers on API 34+ must specify `RECEIVER_EXPORTED` /
  `RECEIVER_NOT_EXPORTED`.
- **Providers**: attribute each to an SDK or app feature (cross-reference
  `02-engineering/04-sdk-inventory.md`); `FileProvider` paths file is part of this inventory.

Exported analysis itself (severity, API 31 explicitness) belongs to
`02-engineering/06-manifest-audit.md`; here you record the facts.

## Step 2 — Background Work Patterns

Detect every scheduling mechanism in use:

```bash
grep -rl --include="*.kt" "WorkManager\|OneTimeWorkRequest\|PeriodicWorkRequest" app/src
grep -rl --include="*.kt" --include="*.java" "JobScheduler\|JobService\|JobInfo" app/src
grep -rl --include="*.kt" --include="*.java" "AlarmManager" app/src
grep -rn --include="*.kt" --include="*.java" "setExactAndAllowWhileIdle\|setExact(\|setAlarmClock" app/src
grep -rl --include="*.kt" --include="*.java" "startForegroundService\|startForeground(" app/src
grep -rl --include="*.kt" --include="*.java" "IntentService" app/src
grep -rn "GcmNetworkManager\|FirebaseJobDispatcher" app/src          # ancient, removal mandatory
```

```powershell
$src = Get-ChildItem -Recurse app\src -Include *.kt,*.java
$src | Where-Object Extension -eq ".kt" | Select-String -List "WorkManager|OneTimeWorkRequest|PeriodicWorkRequest" | Select-Object Path
$src | Select-String -List "JobScheduler|JobService|JobInfo" | Select-Object Path
$src | Select-String -List "AlarmManager" | Select-Object Path
$src | Select-String "setExactAndAllowWhileIdle|setExact\(|setAlarmClock"
$src | Select-String -List "startForegroundService|startForeground\(" | Select-Object Path
$src | Select-String -List "IntentService" | Select-Object Path
$src | Select-String "GcmNetworkManager|FirebaseJobDispatcher"        # ancient, removal mandatory
```

For each hit record: what work it does, trigger (time/network/charging/push), frequency, and
whether it must survive process death. Decision rule for the modern target:

| Need | Modern mechanism |
|---|---|
| Deferrable guaranteed work (sync, upload, cleanup) | WorkManager |
| Exact-time user-facing event (alarm clock, medication reminder) | `AlarmManager.setExactAndAllowWhileIdle` + `USE_EXACT_ALARM` (alarm-app core function) or `SCHEDULE_EXACT_ALARM` (user-grantable, revocable) |
| Ongoing user-visible task (playback, tracking, download) | Foreground service with correct type |
| While-app-visible work | Coroutines in ViewModel scope — no service at all |
| Push-triggered | FCM + WorkManager for the heavy part |

## Step 3 — Android 12–16 Restriction Compliance

The full per-API behavior-change matrix lives in `08-knowledge/android/version-matrix.md` — cite
it for anything UI- or storage-related (edge-to-edge, predictive back, scoped storage, media
permissions). The table below keeps only the **component and background-work** restrictions this
method audits; check each row of Step 1/2 against them:

| API | Restriction (components & background work only) |
|---|---|
| 31 (Android 12) | No foreground-service start from background (narrow exemptions: FCM high priority, exact alarm, user interaction). `android:exported` mandatory on any component with intent filters (full audit: `06-manifest-audit.md` Step 2). Notification trampolines banned (services/receivers cannot `startActivity` from a notification tap). |
| 33 (Android 13) | `POST_NOTIFICATIONS` runtime permission — all notification-posting paths (incl. foreground services) need the request flow. |
| 34 (Android 14) | **Every** foreground service must declare `android:foregroundServiceType` in the manifest AND pass the type to `startForeground()`, AND hold that type's prerequisite permission. Types: `camera`, `connectedDevice`, `dataSync`, `health`, `location`, `mediaPlayback`, `mediaProjection`, `microphone`, `phoneCall`, `remoteMessaging`, `shortService`, `specialUse`, `systemExempted`. Each type also needs `FOREGROUND_SERVICE_<TYPE>` permission (e.g., `FOREGROUND_SERVICE_DATA_SYNC`). `SCHEDULE_EXACT_ALARM` is no longer pre-granted for newly installed apps targeting 34+ — must check `canScheduleExactAlarms()` and route the user to the grant screen. Context-registered receivers need export flags. |
| 35 (Android 15) | `dataSync` and `mediaProcessing` foreground services get a 6-hour-per-24h runtime cap. `BOOT_COMPLETED` receivers cannot launch several FGS types. |
| 36 (Android 16) | Further JobScheduler/FGS quota tightening — verify current behavior-change docs (`08-knowledge/android/version-matrix.md`) before setting targetSdk 36. |

Play policy overlay: `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` and `foregroundServiceType=
specialUse` require declaration forms in Play Console — flag for
`02-engineering/06-manifest-audit.md` and `08-knowledge/stores/policy-landmines.md`.

## Step 4 — Risk Table (legacy pattern → breaks at → migration)

Produce this table with one row per detected legacy pattern; it goes verbatim into the audit:

| Legacy pattern | Breaks / degrades at | Failure mode | Migration | Effort |
|---|---|---|---|---|
| Started `Service` for deferrable work | API 26+ (background execution limits) | `IllegalStateException` on background start; work silently dies | WorkManager | M |
| `IntentService` | Deprecated API 30 | Still runs but unmaintained; background-start crashes | WorkManager / coroutine | S–M |
| FGS without `foregroundServiceType` | **API 34: crash** (`MissingForegroundServiceTypeException`) | Hard crash on `startForeground` | Add type + permission + runtime check | S |
| `setExact` without exact-alarm permission handling | API 31 default-deny → API 34 not pre-granted | `SecurityException` or silent inexact fallback | `canScheduleExactAlarms()` gate + settings intent, or downgrade to WorkManager | S–M |
| Manifest receiver for implicit broadcasts | API 26 | Never fires | Context-register or WorkManager triggers | S |
| Context receiver without export flag | API 34: crash on register | `SecurityException` | Add `RECEIVER_NOT_EXPORTED` flag | S |
| Notification trampoline | API 31 | Tap does nothing | Direct `PendingIntent` to Activity | S |
| Notifications without `POST_NOTIFICATIONS` flow | API 33 | All notifications silently dropped | Permission request flow | S |
| `dataSync` FGS for long syncs | API 35 | Killed at 6h quota | WorkManager with expedited/long-running work | M |
| `GcmNetworkManager`/`FirebaseJobDispatcher` | Already dead (GCM sunset) | Work never runs | WorkManager | M |

## Step 5 — Deep Links, App Links, Widgets, Shortcuts

`$MERGED_MANIFEST` below = the merged-manifest file obtained in Step 1
(`06-manifest-audit.md` Step 0).

```bash
# Deep links & app links in merged manifest
grep -n "android:scheme\|android:host\|autoVerify" "$MERGED_MANIFEST"
# assetlinks verification target:
# https://<host>/.well-known/assetlinks.json must list the release signing cert SHA-256
# Widgets
grep -rn --include="*.kt" --include="*.java" "AppWidgetProvider" app/src; ls app/src/main/res/xml/*widget* 2>/dev/null
# Shortcuts
ls app/src/main/res/xml/shortcuts.xml 2>/dev/null; grep -rn --include="*.kt" "ShortcutManager\|setDynamicShortcuts" app/src
```

```powershell
# Deep links & app links in merged manifest
Select-String -Path $MERGED_MANIFEST -Pattern "android:scheme|android:host|autoVerify"
# Widgets
Get-ChildItem -Recurse app\src -Include *.kt,*.java | Select-String "AppWidgetProvider"
Get-ChildItem app\src\main\res\xml -Filter *widget* -ErrorAction SilentlyContinue
# Shortcuts
Test-Path app\src\main\res\xml\shortcuts.xml
Get-ChildItem -Recurse app\src -Filter *.kt | Select-String "ShortcutManager|setDynamicShortcuts"
```

Record each deep-link URI pattern and which Activity handles it; `autoVerify="true"` without a
reachable, correct `assetlinks.json` = High finding (links fall back to disambiguation dialog).
Widgets and shortcuts are also **design surfaces**: list them for `03-design/02-screen-inventory.md`
so the P5 redesign doesn't orphan them.

## Step 6 — Processes, Startup Initializers, Notification Channels

Three auxiliary runtime surfaces that audits routinely miss:

```bash
# Multi-process: any component with android:process complicates SDK init (Application.onCreate
# runs once per process — guard SDK init with a main-process check)
grep -n "android:process" "$MERGED_MANIFEST"

# androidx.startup initializers (run before Application.onCreate completes)
grep -n "androidx.startup" "$MERGED_MANIFEST"
grep -rn --include="*.kt" "Initializer<" app/src

# Notification channels: every posting path needs a channel on API 26+; inventory them
grep -rn --include="*.kt" --include="*.java" "NotificationChannel(\|createNotificationChannel" app/src
```

```powershell
# Multi-process components
Select-String -Path $MERGED_MANIFEST -Pattern "android:process"

# androidx.startup initializers
Select-String -Path $MERGED_MANIFEST -Pattern "androidx.startup"
Get-ChildItem -Recurse app\src -Filter *.kt | Select-String "Initializer<"

# Notification channels
Get-ChildItem -Recurse app\src -Include *.kt,*.java | Select-String "NotificationChannel\(|createNotificationChannel"
```

Record per channel: id, importance, which feature posts to it, and whether the `POST_NOTIFICATIONS`
request flow (API 33+, Step 3) covers that feature. Orphan channels (created but never posted to)
and channel-less `notify()` calls (notification silently dropped on 26+) are both Low–Medium
findings. If `android:process` is found, verify every SDK init site checks the process name —
double-initialized analytics SDKs double-count events (cross-reference
`02-engineering/04-sdk-inventory.md` init locations).

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Component tables (activities/services/receivers/providers) | `docs/factory/audits/ENGINEERING_AUDIT.md` "Components" section | `06-manifest-audit.md` exported/permission analysis; P5 screen inventory |
| Background-work map (mechanism, trigger, frequency) | ENGINEERING_AUDIT.md | targetSdk-raise plan in `02-engineering/migrations/android-sdk-migration.md` |
| Risk table (Step 4) | ENGINEERING_AUDIT.md, rows promoted to ENG- findings (severity = Critical if "crash at current targetSdk+1", else per failure mode) | Modernization roadmap (P4) ordering; `09-templates/risk-register.md` |
| Exact-alarm / FGS-type policy flags | ENG- findings | `06-manifest-audit.md`; Play declaration forms at P8 (`10-releases/release-workflow.md`) |
| Deep-link/widget/shortcut inventory | ENGINEERING_AUDIT.md + screen inventory | P5 design audit; P7 ASO (links in store campaigns); `15-final-verification.md` smoke flows |

Complete when: every merged-manifest component has a table row with purpose and source attributed,
every background-work code path has a mechanism row, and every legacy pattern has a risk-table row
with a migration target — no "unknown purpose" rows may remain at audit sign-off.
