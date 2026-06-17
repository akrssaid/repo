# Common Android Pitfalls Catalog

Hard-won failure patterns that recur across modernization projects, grouped by where they bite:
Build/Migration, Runtime, UI, Release. Each entry is Symptom / Cause / Fix / Prevention so an
agent can both diagnose an active failure and write the prevention rule into the target app's
roadmap. Consult during P3–P6 and mandatorily during P8 verification
(`01-core/prompts/15-final-verification.md`). Platform behaviors referenced here are stable
history; tool defaults drift — **verify against current official docs** where flagged.

**last reviewed: 2026-06-10**

---

## Build / Migration

### B1 — R8 full mode silently strips reflection-used classes

- **Symptom**: Release build crashes (`ClassNotFoundException`, `NoSuchMethodException`, or Gson
  returning objects with null fields / default values) while debug builds work perfectly. Often
  surfaces only on one API response or one screen.
- **Cause**: R8 full mode (default since AGP 8.0 — verify current default) removes or renames
  anything not statically reachable. Reflection consumers (Gson models, `Class.forName`, manifest
  `<meta-data>` class refs, JNI callbacks) give R8 no static reference.
- **Fix**: Add targeted `-keep` rules for the reflective surface, or better, remove the reflection
  (Gson → kotlinx.serialization per `08-knowledge/android/modern-stack.md`). Decode the stack
  trace with the build's `mapping.txt` first.
- **Prevention**: Test the *release* build of every touched flow before shipping (see R-group
  pitfall RL3); prefer codegen libraries over reflective ones; keep `-keep` rules commented with
  the reason so they can be deleted when the cause is migrated.

### B2 — kapt/KSP double-processing

- **Symptom**: Duplicate-class errors, "symbol already defined", Hilt/Room generating twice,
  inexplicably slow builds after a "migrate to KSP" commit.
- **Cause**: The same processor declared under both `kapt(...)` and `ksp(...)` (often via a
  half-finished migration or a convention plugin still applying `kotlin-kapt`).
- **Fix**: Audit every module: `Select-String -Path **\build.gradle.kts -Pattern 'kapt'`. Remove
  the kapt declaration *and* the `kotlin-kapt` plugin once no processor needs it.
- **Prevention**: Migrate kapt→KSP as one atomic commit per module; CI greps for the string
  `kapt` after the migration milestone closes.

### B3 — Version catalog typos produce misleading errors

- **Symptom**: `Could not resolve all files`, "unknown property `libs.foo`", or Gradle resolving a
  totally different artifact — error messages point at the *usage site*, not the typo in
  `gradle/libs.versions.toml`.
- **Cause**: Catalog keys are translated (`my-lib` → `libs.my.lib`); a dash/dot mismatch, a typo
  in `version.ref`, or a stale catalog cache yields errors far from the real mistake.
- **Fix**: Open `gradle/libs.versions.toml` first whenever a dependency error appears after a
  catalog edit; check the exact key spelling and the `version.ref` target exists; run
  `.\gradlew --refresh-dependencies` if metadata seems stale.
- **Prevention**: One alias naming convention per repo (kebab-case, group-name ordering);
  catalog edits get their own commit so `git bisect` isolates them.

### B4 — AGP namespace vs manifest package mismatch

- **Symptom**: After AGP upgrade: "Namespace not specified", duplicate-R classes, resources
  resolving against the wrong package, or `BuildConfig` import breaking in flavored builds.
- **Cause**: AGP 8 requires `namespace` in `build.gradle.kts` and ignores/forbids `package=` in
  the manifest. Legacy apps often have manifest `package` ≠ `applicationId`, and code imports `R`
  from the old package.
- **Fix**: Set `namespace` to the *old manifest package* (preserving `R`/`BuildConfig` imports),
  keep `applicationId` unchanged (it defines store identity — never change it), delete `package=`
  from the manifest.
- **Prevention**: `02-engineering/migrations/agp-migration.md` step order; rule: `applicationId`
  is immutable for a published app, `namespace` is a code-organization detail.

### B5 — Missing Room migration crashes existing users on update

- **Symptom**: The app works on a fresh install but crashes on launch *for users updating from a
  prior version* with `IllegalStateException: Room cannot verify the data integrity. Looks like
  you've changed schema but forgot to update the version number` — or, worse, the schema changed
  *without* bumping the version and Room silently reads a mismatched table until a query throws.
  Reproduces only when you install the old version first, then the new one over it.
- **Cause**: Any change to a Room `@Entity` (added/renamed/dropped column, new table, changed type
  or index) requires both a bumped `version =` in `@Database` **and** a `Migration` (or
  `AutoMigration`) describing the schema delta. Fresh installs build the latest schema directly, so
  debug-on-emulator testing never hits the gap; only the *upgrade* path runs migrations.
- **Fix**: Provide the migration. Hand-write `Migration(from, to)` with the exact ALTER/CREATE SQL,
  or declare `@AutoMigration` (with an `AutoMigrationSpec` for renames/deletes Room can't infer);
  register it on the database builder. Export schemas (`room.schemaLocation`) so the JSON diff is
  reviewable and migration tests can run against real before/after schemas. **Never ship
  `fallbackToDestructiveMigration()` in production** — it wipes user data on every schema change.
  Full procedure and the migration-test harness live in
  `02-engineering/11-data-migration-safety.md`.
- **Prevention**: Treat every entity edit as a migration ticket; CI fails if the exported schema
  changed without a new migration + migration test; the upgrade path (install old AAB → install
  new) is a mandatory P8 verification step, not the fresh-install path. See
  `02-engineering/11-data-migration-safety.md`.

---

## Runtime

### R1 — Process death state loss

- **Symptom**: App "randomly" loses screen state, crashes with null session/late-init data, or
  restarts at a broken mid-flow screen — typically reported as "happens when I come back to the
  app after a while". Never reproduces in a debugger session.
- **Cause**: Android kills backgrounded processes; on return, the OS restores the back stack but
  all singletons/ViewModels' in-memory state is gone. Code assumes "Application onCreate ran
  recently" or "previous screen populated this object".
- **Fix**: Persist restoration-critical state (`SavedStateHandle`, Room/DataStore); make every
  screen able to self-load from its arguments; route through a real start destination if the
  session is truly gone.
- **Test procedure** (mandatory in P8 verification):

```powershell
# Put app in background (press Home on device), then:
adb shell am kill <package.name>      # kills process but keeps task/back stack
# Relaunch from recents — the app must restore correctly on every screen.
```

- **Prevention**: Treat `SavedStateHandle` + persisted repositories as the source of truth; never
  pass live objects between screens (pass IDs); run the kill test on every new/redesigned screen.

### R2 — Vendor background-killing (aggressive OEMs)

- **Symptom**: WorkManager jobs, FGS, and even alarms never run on specific devices (heavily
  customized OEM ROMs — historically Xiaomi/MIUI, Huawei/EMUI, Oppo, Vivo, OnePlus); reviews say
  "downloads stop when I close the app" only from those brands.
- **Cause**: OEM "battery optimizers" force-stop apps beyond AOSP rules; a force-stopped app gets
  *no* scheduled work until next manual launch. WorkManager guarantees are AOSP guarantees.
- **Fix**: No code fully fixes this. Mitigate: run user-initiated long tasks as a typed foreground
  service *while the user watches*; design jobs to resume idempotently on next launch; optionally
  deep-link users to vendor whitelisting screens (dontkillmyapp.com documents per-vendor paths —
  verify currency before citing).
- **Prevention**: Set expectations in roadmaps: background guarantees are "best effort on OEM
  ROMs"; never promise exact-time background behavior in store listings; build resume-on-open
  into every long-running feature.

### R3 — Exact-alarm permission revocation

- **Symptom**: Reminders/alarms silently stop firing for some users on Android 14+; `SecurityException`
  on `setExact*` for apps targeting 31+ without the grant.
- **Cause**: `SCHEDULE_EXACT_ALARM` (API 31+) is a special access the user/system can revoke, and
  it is **denied by default for newly installed apps targeting 33+** (verify current policy).
  `USE_EXACT_ALARM` bypasses the toggle but is policy-restricted to alarm/calendar apps.
- **Fix**: Check `AlarmManager.canScheduleExactAlarms()` before scheduling; fall back to inexact
  (`setWindow`) or WorkManager when denied; surface an in-context request
  (`ACTION_REQUEST_SCHEDULE_EXACT_ALARM`) only when the feature truly needs exactness.
- **Prevention**: Audit every `setExact` call in P3 manifest/service review
  (`02-engineering/06-manifest-audit.md`); classify each as needs-exact vs tolerates-window.

### R4 — Memory leaks from Activity-context retention

- **Symptom**: OOM crashes after extended use, rotation jank, LeakCanary reports an Activity
  retained after destroy; heap dumps show whole View trees pinned.
- **Cause**: An Activity (or View/Fragment) reference stored anywhere that outlives it — singleton
  fields, static caches, long-lived coroutine/listener registered with Activity context, image
  loaders or DI graphs fed `this` instead of `applicationContext`.
- **Fix**: Identify the retention chain (LeakCanary in debug builds), replace with
  `applicationContext`, lifecycle-scoped coroutines, or unregister in `onDestroy`.
- **Prevention**: Rule: anything with a lifecycle longer than a screen gets application context
  only; LeakCanary in debug builds is part of the factory's modernization definition-of-done
  (`07-checklists/engineering-modernization.md`).

### R5 — WorkManager initialization conflict / double-init

- **Symptom**: One of: `IllegalStateException: WorkManager is already initialized` at startup;
  workers that depend on Hilt/DI inject nothing and `NullPointerException` on first run;
  `WorkManager.getInstance()` throws "not initialized" in a Hilt app; or workers run with a
  default config that ignores a custom `WorkerFactory`. Often appears right after adding `HiltWorker`
  or a custom `Configuration`.
- **Cause**: WorkManager auto-initializes via its bundled `androidx.startup` `InitializationProvider`
  (a manifest `ContentProvider`). If the app *also* provides a custom `Configuration` — required
  for `HiltWorkerFactory` or any custom `WorkerFactory` — there are now two competing init paths.
  The two correct patterns are mutually exclusive and must not be mixed: either (a) let on-demand
  init run by having the `Application` implement `Configuration.Provider`, or (b) keep the default
  provider. Removing the provider node *and* not implementing `Configuration.Provider` yields
  "not initialized"; keeping the provider *and* calling `WorkManager.initialize()` manually yields
  "already initialized".
- **Fix**: For Hilt/custom-factory apps, use **on-demand initialization**: remove the default
  initializer in the manifest —
  `<provider android:name="androidx.startup.InitializationProvider" ...>` with a
  `<meta-data android:name="androidx.work.WorkManagerInitializer" tools:node="remove" />` — and make
  the `Application` implement `Configuration.Provider`, returning the config that installs the
  `HiltWorkerFactory`. Never also call `WorkManager.initialize()` yourself. For apps with no custom
  config, leave the default provider untouched and do nothing.
- **Prevention**: Decide the init strategy once when WorkManager is introduced and document it;
  grep the merged manifest for `WorkManagerInitializer` to confirm exactly one strategy is live
  (merged-manifest read per `02-engineering/06-manifest-audit.md` Step 0); run a worker that
  resolves an injected dependency as a smoke test. Stack rationale in
  `08-knowledge/android/modern-stack.md` (background table).

### R6 — StrictMode disabled, so main-thread I/O ships undetected

- **Symptom**: No local symptom — that is the trap. Production shows scattered ANRs (see
  `08-knowledge/android/play-vitals-performance.md` for the rate that gates visibility), slow cold
  starts, and frame drops on first interaction, none of which reproduce on the developer's fast
  device with a warm cache. The disk/network read blocking the main thread is invisible because
  nothing was watching the main thread.
- **Cause**: StrictMode (`ThreadPolicy` for main-thread disk/network reads, `VmPolicy` for leaked
  closeables/Activities/SQLite cursors) is off by default and was never enabled in the debug
  `Application`. Legacy code paths — `SharedPreferences.getX` on first access, synchronous file
  reads, a `SQLiteOpenHelper` query on the UI thread, image decode on the main thread — run fine on
  a fast device but cause real ANRs on budget hardware in the field.
- **Fix**: Enable StrictMode in `debug` builds only, in `Application.onCreate`, with
  `detectAll().penaltyLog()` (add `penaltyDeath()` once the baseline is clean so new violations fail
  fast). Triage every logged violation: move the offending I/O off the main thread (coroutine +
  `Dispatchers.IO`, DataStore instead of synchronous SharedPreferences — see the persistence row in
  `08-knowledge/android/modern-stack.md`). Never enable `penaltyDeath` in release.
- **Prevention**: StrictMode-in-debug is part of the modernization definition-of-done
  (`07-checklists/engineering-modernization.md`); pair it with the startup/jank measurement in
  `02-engineering/10-performance-audit.md` so main-thread stalls are caught before users feel them.

---

## UI

### U1 — Compose recomposition storms

- **Symptom**: Janky scrolling, hot device, Layout Inspector shows leaf composables recomposing
  hundreds of times; animations stutter on mid-range hardware.
- **Cause**: Unstable parameters (mutable lists, classes from non-Compose modules without a
  stability config), new lambda instances per recomposition capturing changing state, reading a
  frequently-changing `State` high in the tree instead of deferring (`{ value }` lambda form).
- **Fix**: Run the Compose compiler metrics/reports; mark models `@Immutable`/use
  `kotlinx.collections.immutable`; hoist state reads down, pass stable method references; use
  `derivedStateOf` for computed-from-scroll values.
- **Prevention**: Stability configuration file for cross-module models; performance pass with
  Layout Inspector recomposition counts is a P6 acceptance item; never benchmark in debuggable
  builds (Compose is dramatically slower there — judge jank on release builds only).

### U2 — XML↔Compose interop measure/theme drift

- **Symptom**: Embedded Compose content sized 0dp or infinitely tall inside `ScrollView`/XML
  containers; fonts/colors subtly differ between the XML half and Compose half of one screen;
  M2 widgets look "off" next to M3.
- **Cause**: Different measurement contracts (Compose in XML needs bounded constraints), and two
  theming systems (XML theme vs `MaterialTheme`) that do not inherit from each other.
- **Fix**: Give `ComposeView` explicit constraints inside scrollables; bridge themes during
  transition (read XML theme attrs into the Compose theme, or use an accompanist-style theme
  adapter); align typography by defining both from one token source
  (`03-design/04-design-system-extraction.md`).
- **Prevention**: Migrate whole screens, not widgets-within-screens, whenever possible; one
  design-token source of truth feeding both systems during the transition window.

### U3 — Keyboard/IME inset bugs at targetSdk 35+

- **Symptom**: After the targetSdk 35/36 bump: text fields hidden behind the keyboard, bottom
  buttons unreachable while typing, content jumping when the IME animates, `adjustResize`
  apparently ignored.
- **Cause**: Enforced edge-to-edge changes inset delivery; legacy `windowSoftInputMode` tricks and
  manual padding hacks fight the new model instead of consuming `WindowInsets.ime`.
- **Fix**: In Compose: `Modifier.imePadding()` on the input container (and `navigationBarsPadding`
  where stacked), `contentWindowInsets` on Scaffold; remove ad-hoc bottom paddings. In Views:
  `ViewCompat.setOnApplyWindowInsetsListener` and `WindowCompat.setDecorFitsSystemWindows(window, false)`
  handled once, at the root.
- **Prevention**: P8 verification includes a keyboard pass on every input screen (open IME,
  rotate, dismiss, multi-line growth); insets handled at root + per-screen `imePadding`, never
  with hardcoded paddings.

### U4 — System-bar contrast after edge-to-edge

- **Symptom**: White status-bar icons on white app background (or the inverse) after the
  edge-to-edge migration; 3-button nav bar floating on a translucent scrim that clashes with the
  theme; screenshots for the store look broken.
- **Cause**: Edge-to-edge removes the opaque system-bar backgrounds the app relied on; icon
  appearance (`isAppearanceLightStatusBars`) no longer auto-matches content, and dark theme
  doubles the matrix.
- **Fix**: Set system-bar icon appearance from the active theme (e.g. `enableEdgeToEdge()` with
  explicit `SystemBarStyle`, or `WindowInsetsControllerCompat`); verify in light theme, dark
  theme, gesture nav, and 3-button nav — four combinations, every top-level screen.
- **Prevention**: Add the four-combination sweep to `07-checklists/design-modernization.md`
  acceptance; design screens assuming content draws under bars from day one.

### U5 — Double splash (system SplashScreen API + legacy splash Activity)

- **Symptom**: On Android 12+ users see **two** splashes in sequence: the system icon splash, then
  a legacy branded splash Activity/screen — a visible flash, a theme/color jump between them, and a
  measurably slower time-to-content. Sometimes the system splash shows the wrong (default) icon or
  the app "flashes white" before the real content.
- **Cause**: Android 12 (API 31) introduced the system `SplashScreen` (shown for *every* app
  launch, always). A legacy app that still has its own splash Activity, or a hand-rolled splash
  composable/Fragment, now stacks on top of the system one. Compounding causes: the launch theme's
  `windowBackground` doesn't match (white flash), or the app holds the splash open with
  `setKeepOnScreenCondition` far longer than first-frame readiness (vanity delay, violates the
  core-task-first rule in `08-knowledge/design/mobile-ux-principles.md`).
- **Fix**: Adopt the `androidx.core:core-splashscreen` API as the *only* splash: call
  `installSplashScreen()` in the launch Activity before `setContentView`/`setContent`, set the
  themed icon/background via the `Theme.SplashScreen` attributes, and **delete the legacy splash
  Activity and its manifest `LAUNCHER` entry**. Keep the splash on-screen only until the first real
  frame is ready (a short `setKeepOnScreenCondition` tied to genuine load state, not a fixed timer).
  This is owned end-to-end by prompt 09's procedure, sourced from `03-design/07-splash-redesign.md`.
- **Prevention**: One launch entry point; no splash Activity in modern apps; the splash dismisses
  the instant content is ready, never on a timer. Verify on an API 31+ device that exactly one
  splash appears with no color jump (a P6/P8 design-modernization acceptance item).

---

## Release

### RL1 — Forgetting the mapping.txt upload

- **Symptom**: Play Console crash reports show obfuscated frames (`a.b.c(Unknown Source)`);
  triage of production crashes becomes guesswork weeks later.
- **Cause**: R8 mapping file for that exact build never uploaded; app bundles normally embed it
  automatically, but custom CI, APK distribution, or stripped bundle settings skip it.
- **Fix**: Retroactively upload `app/build/outputs/mapping/release/mapping.txt` for the affected
  version in Play Console (App bundle explorer → version → downloads/deobfuscation), and to
  Crashlytics if used. Archive mapping files per release, forever.
- **Prevention**: Release workflow (`10-releases/release-workflow.md`) archives
  `mapping.txt` + native debug symbols as a gated step; verification checks a symbolicated test
  crash for each new release.

### RL2 — versionCode collision across tracks

- **Symptom**: Play Console rejects an upload: "Version code X has already been used"; or a
  internal-track build blocks the production rollout because it shares/precedes a code.
- **Cause**: versionCode must be strictly increasing per artifact and is shared across all tracks;
  parallel branches/CI jobs minting codes independently (or manual edits) collide.
- **Fix**: Bump past the highest code ever uploaded (any track) and re-build; never reuse codes.
- **Prevention**: Single monotonic source for versionCode (CI build number or dated scheme) per
  `10-releases/app-versioning-policy.md`; release checklist records the code in
  `docs/factory/CHANGELOG.md` before upload.

### RL3 — Testing only debug builds

- **Symptom**: Crash/blank-screen reports flood in right after release; nothing reproduces locally.
- **Cause**: Debug builds skip R8, use debug signing/network config, disable some optimizations —
  release-only failures (see B1) are invisible. Debug-only testing is the root enabler of half
  this catalog.
- **Fix**: Build and install the actual release variant (`.\gradlew assembleRelease` or bundle +
  `bundletool install-apks`) and re-test the core flows; symbolicate any crash via `mapping.txt`.
- **Prevention**: `07-checklists/release-readiness.md` requires the full core-flow pass on the
  *release* build, on a physical device, before any track upload. No exceptions.

### RL4 — Locale pseudo-testing skipped

- **Symptom**: Truncated/overlapping text in German/Russian, broken RTL mirroring in Arabic,
  hardcoded English strings discovered by one-star reviews in localized markets.
- **Cause**: UI only ever viewed in English; long-word languages run 30–40% wider; RTL flips
  layout direction; hardcoded strings bypass the resource system silently.
- **Fix**: Enable pseudo-locales and sweep every screen:

```powershell
adb shell settings put system system_locales en-XA,ar-XB   # pseudo-accented + pseudo-RTL
# Build with pseudoLocalesEnabled = true on the debug build type.
```

  Fix overflow with proper constraints, ellipsis policy decisions, and `start`/`end` (never
  `left`/`right`) attributes.
- **Prevention**: Lint rule `HardcodedText` set to error; pseudo-locale sweep is a P6/P8 checklist
  item; localization workflow in `04-aso/workflows/localization.md` assumes resources are clean.

---

Cross-references: stack doctrine in `08-knowledge/android/modern-stack.md`, API-level triggers in
`08-knowledge/android/version-matrix.md`, release gates in `07-checklists/release-readiness.md`.
