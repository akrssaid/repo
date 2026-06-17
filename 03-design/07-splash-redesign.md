# Splash Screen Modernization Method

This document defines the method for modernizing a target app's launch experience using the
Android 12+ SplashScreen API and the `androidx.core:core-splashscreen` compat library. It covers
the API reality on modern Android, legacy-splash detection and removal, exact implementation steps
with dimensions, and a measurement and pitfall protocol.

**Ownership (D12).** The splash is owned by **prompt 09's redesign-implementation procedure**
(`01-core/prompts/09-ui-implementation.md`), which sources a **dedicated splash ticket** from this
method. That ticket carries an `IMPL-NNN` ID, appears in the IMPLEMENTATION handover ticket table
(`05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md`), and is closed and verified like any other P6
ticket — the design-modernization gate (`07-checklists/design-modernization.md`) checks it.
Prompt 10 does **not** own the splash; it only produces the launcher icon
(`03-design/06-icon-redesign.md`), from which the splash icon is derived. This file is the method
the splash ticket executes; conventions (IDs, scales, paths) come from `01-core/CONVENTIONS.md`.

---

## 1. The Android 12+ Reality

On Android 12 (API 31) and later, **the system always shows a splash screen** on cold and warm
starts of any activity launched from the launcher: the app's launcher icon centered on the window
background color, with an automatic zoom-out exit. You cannot opt out. Consequences:

1. Any **legacy custom splash Activity** (a `SplashActivity` that shows a logo, sleeps or loads,
   then starts `MainActivity`) now produces a **double splash**: system splash → your splash →
   home screen. This is the single most common launch-experience defect in apps being modernized
   and is always at least a High design finding.
2. The only sanctioned customization path is theming the system splash (background color, icon,
   optional branding) and controlling its dismissal — via the compat library so behavior is
   consistent back to minSdk.

### 1.1 Legacy splash detection procedure

1. Find launcher entry and splash suspects:

   ```powershell
   Select-String -Path app\src\main\AndroidManifest.xml -Pattern "LAUNCHER" -Context 6
   Get-ChildItem -Recurse app\src -Include "*Splash*","*Launch*" | Select-Object FullName
   ```

2. Classify what the launcher Activity does:

   | Pattern found | Verdict |
   |---|---|
   | Dedicated `SplashActivity` with delay (`Handler.postDelayed`, `delay()`) then `startActivity` | Legacy splash — remove |
   | Launcher theme with `android:windowBackground` set to a logo drawable | Legacy "themed splash" — migrate to SplashScreen theme attributes |
   | Splash that performs real init (config fetch, DB migration, auth check) | Merge: move work behind `setKeepOnScreenCondition` (Section 3) |
   | No custom splash; plain cold start | Clean — implement Section 2 directly |

3. Record the verdict in `docs/factory/audits/DESIGN_AUDIT.md` with a DES-xxx finding.

### 1.2 Removal / merge strategy

- **Pure-delay splash**: delete the Activity, point the LAUNCHER intent-filter at the real main
  Activity, delete the splash layout/theme, and add a manifest `<activity-alias>` only if the old
  splash class name is referenced by shortcuts or external intents (keep the alias targeting the
  main Activity to avoid breaking pinned shortcuts).
- **Working splash**: move its initialization into the main Activity's ViewModel (or an
  `App Startup` initializer), expose an `isReady: StateFlow<Boolean>`, and hold the system splash
  with `setKeepOnScreenCondition { !viewModel.isReady.value }`.
- **Deep links**: re-test every deep link after removal — links that previously entered through
  `SplashActivity` must resolve to the same destination.

---

## 2. Implementation with `androidx.core:core-splashscreen`

The compat library backports themed splash behavior to API 23+ and is the required path even if
minSdk ≥ 31, because it gives a single API surface for exit animations and keep-on-screen logic.

1. **Dependency** (`gradle/libs.versions.toml` + module build file):

   ```kotlin
   implementation("androidx.core:core-splashscreen:1.2.0") // verify latest stable
   ```

2. **Splash theme** in `res/values/themes.xml` (and a `values-night` twin so the splash matches
   dark mode — see pitfall table):

   ```xml
   <style name="Theme.App.Starting" parent="Theme.SplashScreen">
       <item name="windowSplashScreenBackground">@color/splash_background</item>
       <item name="windowSplashScreenAnimatedIcon">@drawable/splash_icon</item>
       <!-- Optional, API 31+ only, drawn behind the icon: -->
       <item name="windowSplashScreenIconBackgroundColor">@color/splash_icon_bg</item>
       <item name="postSplashScreenTheme">@style/Theme.App</item>
   </style>
   ```

3. **Manifest**: set the theme on the launcher Activity (or application):

   ```xml
   <activity android:name=".MainActivity" android:theme="@style/Theme.App.Starting" ...>
   ```

4. **Activity**: call `installSplashScreen()` **before** `super.onCreate` semantics matter — it
   must run before `setContentView`/`setContent`, and the canonical position is the first line:

   ```kotlin
   class MainActivity : ComponentActivity() {
       private val viewModel: MainViewModel by viewModels()

       override fun onCreate(savedInstanceState: Bundle?) {
           val splashScreen = installSplashScreen()
           super.onCreate(savedInstanceState)
           splashScreen.setKeepOnScreenCondition { !viewModel.isReady.value }
           setContent { AppTheme { AppNavHost() } }
       }
   }
   ```

   `postSplashScreenTheme` restores the real app theme after the splash, so the starting theme
   never leaks into normal UI.

### 2.1 Icon dimension rules (exact)

The splash icon drawable occupies a **108 dp** circle slot, but the asset rules are stated at the
canonical 4x scale used by the docs and tooling:

| Asset type | Canvas | Visible/safe content |
|---|---|---|
| Icon WITH its own background (`windowSplashScreenIconBackgroundColor` unused) | 240×240 dp | content within a 160 dp diameter circle |
| Icon WITHOUT background (system draws background color behind it) | 288×288 dp | content within a 192 dp diameter circle |
| Practical rule for the common case | 288 dp canvas | keep the glyph inside the central ~240 dp; critical detail inside 192 dp circle |

In practice: reuse `@drawable/ic_launcher_foreground` from `03-design/06-icon-redesign.md` only if
its glyph already sits comfortably in the adaptive-icon 66 dp safe zone — the same proportional
margin maps onto the splash circle. If the foreground fills its canvas, author a dedicated
`splash_icon.xml` with extra padding; an edge-to-edge drawable WILL be cropped to the circle.
Animated icons (`AnimatedVectorDrawable`) are allowed; keep total animation ≤ 1000 ms.

### 2.2 Reduced-motion fallback (animated splash icons)

If the splash uses an animated icon or a custom exit animation, it MUST honor the system
reduced-motion preference, per the reduced-motion fallback rule in `03-design/10-motion-design.md`
(motion vocabulary lives in `03-design/05-material3-modernization.md`'s `material3-essentials`
reference). When "Remove animations" is enabled (Settings → Accessibility → Remove animations,
backed by `Settings.Global.ANIMATOR_DURATION_SCALE = 0`), the animated splash and any custom exit
must degrade to a **static icon with the default system fade** — no spin/morph/scale.

Detect and branch:

```kotlin
val animScale = Settings.Global.getFloat(
    contentResolver, Settings.Global.ANIMATOR_DURATION_SCALE, 1f
)
val reduceMotion = animScale == 0f
```

- `windowSplashScreenAnimatedIcon` may stay an `AnimatedVectorDrawable`; when `reduceMotion` is
  true, do not add a `setOnExitAnimationListener` (let the default static fade run) and prefer a
  non-animated `splash_icon` resource if the AVD's first frame is not a complete static glyph.
- This is the splash instance of the motion-pipeline reduced-motion contract (D14); the IMPL
  schema's §Motion Specification records the splash row with its reduced-motion fallback, and the
  splash ticket's `Motion:` field names it.
- Verify on a device/emulator with "Remove animations" ON: the splash shows a still icon and exits
  with a plain fade; nothing animates.

---

## 3. Keep-on-Screen and Exit

### 3.1 `setKeepOnScreenCondition`

Use it to hold the splash exactly as long as first-frame-critical loading takes (theme/config,
session restore, DB migration). The lambda is evaluated on every frame; make it cheap.

**Anti-pattern warning — never add artificial delay.** The splash is dead time. Holding it for
branding (`delay(2000)` to "show the logo") is a design defect, not a feature: it inflates
measured cold-start, hurts retention, and on Android 12+ users already saw the system splash.
Hard rule: keep-on-screen duration beyond real readiness must be **0 ms**, and total splash
visibility should stay under ~1 s on a mid-range device. If init legitimately exceeds ~2 s, do
not hold the splash — release it and show the in-app loading state from
`03-design/08-component-library.md` (AppLoadingState), which can communicate progress.

### 3.2 Branded exit animation

`setOnExitAnimationListener` hands you the `splashScreenViewProvider` to animate out (slide, fade,
icon morph into an in-app element). Guidance:

- Keep it ≤ 300 ms; call `splashScreenViewProvider.remove()` in the end-listener or the splash
  view leaks over your UI.
- Respect `splashScreenViewProvider.iconAnimationStartMillis`/`iconAnimationDurationMillis` to
  avoid cutting a running icon animation.
- Setting an exit listener disables the default fade — if you add one, you own the entire exit.
- Skip custom exits entirely unless the design handover (`05-handover/DESIGN_HANDOVER_SCHEMA.md`)
  explicitly specifies one; default system exit is correct for most apps.

---

## 4. Cold-Start Measurement

Measure before and after the splash work; record both in `docs/factory/reports/`.

```powershell
adb shell am force-stop com.example.app
adb shell am start -W -n com.example.app/.MainActivity
```

Read the output:

- `TotalTime` — process start to first frame of the activity: the primary cold-start metric.
- `WaitTime` — includes system overhead; report it but compare on TotalTime.
- Run 5 iterations (force-stop between each), discard the first (disk cache warm-up), report the
  median. Warm start: press home (do not force-stop), relaunch, measure again.
- Regression gate: the modernized splash must not increase median cold TotalTime by more than
  100 ms versus baseline; expected outcome after removing a legacy splash Activity is a large
  improvement (the entire old delay disappears).

---

## 5. Common Pitfalls

| Pitfall | Symptom | Cause | Fix |
|---|---|---|---|
| Double splash | System splash, then a second logo screen | Legacy `SplashActivity` still in manifest | Section 1.2 removal/merge |
| Stretched/cropped icon | Glyph touches or overflows the circle, looks zoomed | Full-bleed drawable used as `windowSplashScreenAnimatedIcon` | Re-author with 288 dp canvas / 192 dp safe circle (Section 2.1) |
| White flash after splash | Brief white frame between splash and first UI in dark mode | No `values-night` splash theme, or `postSplashScreenTheme` mismatch with the runtime theme | Provide night variant of `splash_background`; align post theme with `Theme.App` day/night |
| Splash on warm starts feels heavy | Splash icon shows on every task re-entry | System behavior on 12+ for warm starts is expected; but keep-on-screen logic re-holding it is not | Ensure `isReady` stays true after first init; never reset it on configuration change |
| Splash never dismisses | App hangs on splash | `setKeepOnScreenCondition` lambda never flips (init failure swallowed) | Add a timeout/failure path that sets ready=true and routes to AppErrorState |
| Old icon on splash after icon redesign | Splash shows stale logo | Splash drawable hardcoded to old asset | Point `windowSplashScreenAnimatedIcon` at the redesigned glyph from `03-design/06-icon-redesign.md` |

### Acceptance checklist

- [ ] Splash work is tracked as a dedicated `IMPL-NNN` ticket sourced from this method (D12), present in the IMPLEMENTATION handover ticket table
- [ ] No custom splash Activity remains (or documented justification + alias for shortcuts)
- [ ] `core-splashscreen` theme implemented with day and night variants
- [ ] `installSplashScreen()` is the first call in the launcher Activity's `onCreate`
- [ ] Splash icon respects 288/192 dp rules; verified visually on an API 31+ emulator and an API 26 device profile
- [ ] If animated: reduced-motion fallback verified with "Remove animations" ON — static icon, default fade (§2.2, per `03-design/10-motion-design.md`); recorded in the IMPL §Motion Specification and the ticket's `Motion:` field
- [ ] No artificial hold; keep-on-screen tied only to real readiness with a failure timeout
- [ ] Cold-start medians measured pre/post and recorded; no >100 ms regression
- [ ] All six pitfalls explicitly checked; findings closed in `docs/factory/audits/DESIGN_AUDIT.md`
