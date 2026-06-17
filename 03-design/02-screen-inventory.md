# Screen Inventory Method

This document defines how to build a complete, verified inventory of every screen in a target
Android app. The inventory is the scope contract for the design audit
(`03-design/01-design-audit.md`), screenshot capture (`03-design/03-screenshot-capture.md`), and
the redesign handover — anything not on the inventory does not get audited or redesigned, so
completeness is the whole point. Output is an instance of `09-templates/screen-inventory.md` saved
to the canonical path `docs/factory/audits/SCREEN_INVENTORY.md` (a **separate file**, never embedded
in the design audit — path per `01-core/CONVENTIONS.md` §Canonical paths).

This inventory is consumed by three prompts and must reconcile with each: it scopes the design audit
(`01-core/prompts/05-design-audit.md`), feeds the design handover's Screen Inventory section with its
`States present` column (`01-core/prompts/06-design-handover.md`), and supplies the device/orientation
matrix the adaptive-layout work reads (`01-core/prompts/11-screenshot-generation.md` and
`03-design/11-adaptive-layout.md`). IDs, scales, and paths here are the canonical ones from
`01-core/CONVENTIONS.md`.

## Definition of "Screen"

A screen is any full-window or near-full-window UI surface a user can reach: activities, fragments
acting as destinations, Compose navigation destinations, full-screen dialogs, bottom sheets that
host a task (not mere menus), paywalls, interstitial/app-open ad wrappers, and onboarding pages.
Small dialogs (confirm/alert) are recorded as **states of their parent screen**, not separate
SCR-IDs, unless they host a multi-step task.

## Step 1 — Static Discovery (from code)

Run all of the following from the target repo root. Use ripgrep (`rg`); it is what the Grep tool
wraps and the patterns below are written for it.

### 1a. Manifest activities

```powershell
rg -n "<activity" --glob "**/AndroidManifest.xml" -A 2
```

Capture every `android:name`. Note `android:exported`, `android:theme`, and which activity owns
`MAIN/LAUNCHER`. Activities from manifest merger (libraries, ad SDKs) that ship UI (e.g.,
`com.google.android.gms.ads.AdActivity`, OSS license menus) are recorded but flagged
`third-party` — they are audited only for placement/configuration, not redesigned. To see the
fully merged manifest from the built APK:

```powershell
aapt2 dump xmltree app\build\outputs\apk\debug\app-debug.apk --file AndroidManifest.xml
```

### 1b. Navigation graphs (XML / Navigation Component)

```powershell
rg -n "<fragment|<dialog|<activity" --glob "**/res/navigation/*.xml"
rg -n "app:startDestination" --glob "**/res/navigation/*.xml"
```

Each `<fragment>`/`<dialog>` destination is a screen candidate. Record the graph file and
destination id as the entry-point reference.

### 1c. Compose NavHost destinations

```powershell
rg -n "NavHost\(|NavHostController" --type kotlin
rg -n "composable\(|composable<" --type kotlin
rg -n "dialog\(|bottomSheet\(" --type kotlin
rg -n "navigate\(" --type kotlin
```

`composable("route")` / `composable<RouteType>` calls enumerate destinations; `navigate(` calls
reveal edges (entry points). Also check for multiple NavHosts (nested graphs, bottom-nav tabs):

```powershell
rg -c "NavHost\(" --type kotlin
```

### 1d. Fragment enumeration (non-NavComponent apps)

```powershell
rg -n "class \w+ ?: ?(Fragment|DialogFragment|BottomSheetDialogFragment)" --type kotlin
rg -n "extends (Fragment|DialogFragment|BottomSheetDialogFragment)" --type java
rg -n "FragmentTransaction|supportFragmentManager\.beginTransaction" --type-add "src:*.{kt,java}" -t src
```

Cross-reference transactions to find which fragments are actually reachable and from where.

### 1e. Monetization and system surfaces

```powershell
rg -n "AdView|InterstitialAd|RewardedAd|AppOpenAd|NativeAd|MaxAdView|BannerView" --type-add "src:*.{kt,java}" -t src
rg -n "BillingClient|launchBillingFlow|paywall|Paywall" --type-add "src:*.{kt,java}" -t src
rg -n "ReviewManager|requestReviewFlow" --type-add "src:*.{kt,java}" -t src
```

Map each hit to the screen that hosts it — this fills the "monetization surfaces" column.

## Step 2 — Dynamic Discovery (running app walk)

Static discovery misses remotely-configured screens, conditional onboarding, and dead code. Walk
the running app to confirm reachability and find surprises.

1. Install and launch the debug build on the audit emulator (setup:
   `03-design/03-screenshot-capture.md`):

   ```powershell
   adb install -r app\build\outputs\apk\debug\app-debug.apk
   adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1
   ```

2. Walk every visible navigation affordance breadth-first: bottom nav, drawer, toolbar menus,
   settings, long-press menus, list-item taps. At each new surface, log the current activity:

   ```powershell
   adb shell "dumpsys activity activities | grep topResumedActivity"
   ```

   For fragment/Compose-level identification, correlate what you see with the static list; if
   ambiguous, check the window/compose hierarchy:

   ```powershell
   adb shell dumpsys activity top
   ```

3. Trigger conditional screens deliberately: first-run (clear data first —
   `adb shell pm clear <package>`), paywall (tap premium features), error states (airplane mode:
   `adb shell cmd connectivity airplane-mode enable` / `disable`), notifications-driven screens
   (deep links: `adb shell am start -a android.intent.action.VIEW -d "<scheme>://<path>"`).
4. Any screen reached dynamically but absent from static discovery gets added with a note on how
   it is constructed (server-driven, library, WebView).

## Step 3 — Screen Records

One row per screen in the inventory table. Required columns:

| Column | Content | Rules |
|---|---|---|
| SCR-ID | `SCR-` + zero-padded 3 digits | Assigned once, never renumbered; launcher screen is SCR-001 |
| Name | Human name (e.g., "Document list") | Stable; used in all findings and handovers |
| Class / route | FQCN or Compose route | The code anchor for implementers |
| Entry points | Where users arrive from (SCR-IDs, deep links, notifications) | "unknown" is not allowed — verify or mark unreachable |
| Purpose | One sentence: the user task this screen serves | |
| Toolkit | `XML`, `Compose`, or `Mixed` | Determined per screen, not per app; drives which migration path applies in `03-design/05-material3-modernization.md` |
| Traffic priority | H / M / L | H = on the core-task path or first session; M = regular but secondary; L = rare (settings, about, licenses) |
| Monetization | Surfaces present: `banner`, `interstitial-entry`, `native`, `paywall`, `upsell`, or `—` | From step 1e + dynamic walk |
| States to capture | Subset of: `default`, `empty`, `loading`, `error`, `offline`, `dark` | `default` and `dark` ALWAYS; `offline` whenever the screen does network I/O (distinct from `error`: offline = no connectivity, error = request failed while connected); others if the screen has them. This column is the capture contract for `03-design/03-screenshot-capture.md` and becomes the handover's `States present` column |
| Adaptive / orientation | `phone-only`, `responsive`, or `tablet+`; plus orientation: `portrait`, `landscape`, `both` | Whether the screen reflows for larger windows or rotation; `tablet+`/`landscape`/`both` screens drive the extra captures in `03-design/03-screenshot-capture.md` and the adaptive specs in `03-design/11-adaptive-layout.md`. Default `phone-only` / `portrait` unless the layout demonstrably adapts |
| Notes | third-party flag, A/B variants, WebView content, etc. | |

Example row:

| SCR-ID | Name | Class / route | Entry points | Purpose | Toolkit | Traffic | Monetization | States | Adaptive / orient. | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| SCR-003 | Document list | `DocListFragment` (nav_main.xml) | SCR-001 bottom nav, deep link `docs://list` | Browse and open saved documents | XML | H | banner | default, empty, loading, error, offline, dark | responsive / both | empty state currently blank; two-pane on tablet |

## Step 4 — Coverage Verification

The inventory is complete only when all of these reconcile. Run the checks and record the result
in the inventory's Coverage section.

1. **Manifest check**: every first-party `<activity>` from step 1a maps to ≥ 1 SCR-ID or is
   explicitly listed as `not user-reachable` (with evidence: no launcher intent, no `startActivity`
   call found — `rg -n "StartActivity|startActivity\(Intent\(.*<Name>"`).
2. **Nav graph check**: every XML destination and every `composable(` route maps to an SCR-ID.
   Count match:

   ```powershell
   rg -c "composable\(" --type kotlin
   rg -c "<fragment|<dialog" --glob "**/res/navigation/*.xml"
   ```

3. **Dynamic check**: every screen seen during the walk is in the table.
4. **Monetization check**: every SDK surface from step 1e is attributed to a screen.
5. **Dead-screen list**: destinations that exist in code but are unreachable are listed separately
   — they are modernization-debt input for `docs/factory/audits/ENGINEERING_AUDIT.md`, not audit
   scope.

A typical utility app yields 8–25 screens. If you find fewer than 6, suspect missed dialogs,
onboarding, or paywalls and re-run step 2; if more than 40, confirm you are not recording trivial
confirm-dialogs as screens.

## Output

1. Instantiate `09-templates/screen-inventory.md` with the table, coverage results, and dead-screen
   list; save to `docs/factory/audits/SCREEN_INVENTORY.md` in the target repo.
2. Update `docs/factory/PROJECT_STATE.md` (screen count, toolkit split XML/Compose/Mixed, count of
   monetization surfaces) and append a `docs/factory/CHANGELOG.md` entry, per the consuming
   prompt's Documentation Update Rules (`01-core/prompts/05-design-audit.md`).

## Maintenance Rule

The inventory is append-only during a project: screens discovered later (e.g., during capture or
implementation) get the next free SCR-ID and a changelog note. If a redesign removes a screen, mark
its row `Status: removed in redesign vN` — never delete rows, because findings and screenshots
reference SCR-IDs permanently.
