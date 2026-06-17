# Modern Android Stack (2026 Reference)

The factory's opinionated stack: for each layer, the chosen technology, why, and the legacy
technology it replaces in target apps. This is doctrine for P3 (Engineering Audit — gap analysis
is measured against this stack) and P4 (Modernization — migrations move apps toward it). Specific
versions live in `08-knowledge/android/version-matrix.md` and drift; the *choices* below are
stable but **verify against current official docs** before pinning any artifact version.

**last reviewed: 2026-06-10**

---

## The Stack at a Glance

| Layer | Choice | Replaces |
|---|---|---|
| UI | Jetpack Compose + Material 3 | XML layouts + MaterialComponents (M2) |
| DI | Hilt | Manual DI, raw Dagger, Koin |
| Async | Coroutines + Flow | RxJava, AsyncTask, LiveData-as-stream |
| Persistence | Room + DataStore | Raw SQLite, SharedPreferences |
| Background | WorkManager (mostly) | Background Services, JobScheduler, AlarmManager |
| Networking | Retrofit/OkHttp + kotlinx.serialization | Volley, HttpURLConnection, Gson |
| Images | Coil | Glide, Picasso |
| Navigation | Compose Navigation, type-safe routes | Fragment transactions, XML nav graphs |
| Build | Version catalogs + convention plugins + KSP | Hardcoded deps, buildSrc sprawl, kapt |

---

## Layer-by-Layer Rationale

### UI — Jetpack Compose + Material 3 (← XML + MaterialComponents)

- Declarative UI eliminates the View-state synchronization bugs that dominate legacy crash logs;
  state flows one way, UI is a function of it.
- M3 brings dynamic color, tonal surfaces, and current platform look for free — the design phases
  (`03-design/05-material3-modernization.md`) assume it.
- Interop is first-class (`ComposeView` / `AndroidView`), so migration is screen-by-screen, not
  big-bang. Keep XML only for screens scheduled for deletion.
- Watch for recomposition pitfalls — see `08-knowledge/android/common-pitfalls.md` (UI section).

### DI — Hilt (← Dagger / manual / Koin)

- Hilt is Dagger with the Android wiring (components, scopes, `@AndroidEntryPoint`) standardized:
  compile-time validation, no service-locator stringliness, official Jetpack integration
  (`hiltViewModel()`, WorkManager `HiltWorker`).
- Raw Dagger: keep only if an existing setup is large and healthy — Hilt sits on top of Dagger, so
  incremental adoption is possible; do not hand-write new components.
- **Koin tradeoff note**: Koin is lighter to learn and has no annotation processing, but resolves
  at runtime — wiring errors surface as runtime crashes instead of build failures, and it lacks
  Hilt's first-party Jetpack hooks. Acceptable to *leave in place* in a small, working codebase;
  never *introduce* it in factory work. Manual DI is acceptable only in single-module toy apps.

### Async — Coroutines + Flow (← RxJava / AsyncTask / LiveData-as-stream)

- Structured concurrency ties work to lifecycles (`viewModelScope`, `lifecycleScope`) — the class
  of "callback fires after Activity death" crashes disappears.
- `Flow`/`StateFlow` cover the reactive cases RxJava was used for, with far smaller API surface;
  Room, DataStore, Retrofit, and Compose all speak coroutines natively.
- AsyncTask is deprecated and broken under modern process management — always migrate on sight.
- LiveData: fine as a leaf in old ViewModels, but stop using it as a *stream abstraction*; new
  code exposes `StateFlow` and collects with `collectAsStateWithLifecycle()` in Compose.
- RxJava migration is mechanical for most operators (`Single`→`suspend fun`, `Observable`→`Flow`);
  migrate per-feature, bridge with `kotlinx-coroutines-rx3` during transition.

### Persistence — Room + DataStore (← raw SQLite + SharedPreferences)

- Room: compile-time-verified SQL, migration framework, `Flow` query observation, KSP processing.
  Raw `SQLiteOpenHelper` code is a defect magnet (no schema verification, hand-rolled cursors).
- DataStore (Preferences or Proto) replaces SharedPreferences: async by design (no main-thread
  disk hits / ANRs), transactional, typed with Proto. SharedPreferences also has known data-loss
  behavior under process kill mid-commit.
- Migration order: SharedPreferences→DataStore is low-risk and ships a one-time migration helper;
  SQLite→Room can wrap existing schemas (`@SkipQueryVerification` temporarily) before cleanup.

### Background — WorkManager (← Services / JobScheduler / AlarmManager)

WorkManager is the default, but "everything is WorkManager" is wrong. Decision table:

| Need | Right tool | Why |
|---|---|---|
| Deferrable, guaranteed work (sync, upload, cleanup) | WorkManager | Survives reboot/process death; respects Doze; constraint API |
| User-visible ongoing task (download, playback, recording) | Foreground Service with declared FGS type | API 34+ requires type + Play-policy-matching use case |
| Work only while UI is visible | Coroutine in `viewModelScope`/`lifecycleScope` | No scheduler needed; dies with the screen, correctly |
| Exact-time user-facing event (alarm clock, reminder at 09:00) | AlarmManager `setExactAndAllowWhileIdle` + `SCHEDULE_EXACT_ALARM`/`USE_EXACT_ALARM` | WorkManager is inexact by design; exact alarms are policy-gated — see pitfalls doc |
| Inexact periodic background refresh | WorkManager `PeriodicWorkRequest` (≥15 min) | JobScheduler directly buys nothing over this |
| Raw JobScheduler | Almost never | WorkManager wraps it; keep only in working legacy code untouched by the roadmap |

Vendor background-killing (aggressive Chinese-OEM task killers) caps what any scheduler can
promise — see `08-knowledge/android/common-pitfalls.md` (Runtime section).

### Networking — Retrofit/OkHttp + kotlinx.serialization (← Gson et al.)

- Retrofit + OkHttp remains the industry default: interceptors, connection pooling, suspend
  function support, certificate pinning, mature ecosystem.
- **kotlinx.serialization over Gson**: Gson uses reflection — it silently constructs objects
  without running constructors, ignores Kotlin nullability (a `val x: String` can arrive null),
  and requires fragile ProGuard/R8 keep rules (a classic release-only crash; see pitfalls doc).
  kotlinx.serialization is compile-time generated, Kotlin-null-safe, and needs no keep rules.
  Moshi (with codegen) is an acceptable existing state, not a target.
- Ktor Client is a reasonable alternative when KMP is in play; for Android-only factory targets,
  Retrofit stays the default for ecosystem leverage.

### Images — Coil (← Glide / Picasso)

- Coil is Kotlin-first, coroutine-based, Compose-native (`AsyncImage`), small, and actively
  maintained. Glide works but is Java/annotation-processor era with clunky Compose interop;
  Picasso is effectively in maintenance — migrate on sight.
- Migration is usually a find-and-replace plus removal of `kapt` for Glide's compiler.

### Navigation — Compose Navigation with type-safe routes

- Use the type-safe (serializable route classes) API — string route templates with manual argument
  encoding were the largest source of navigation bugs in early Compose apps.
- Single-Activity architecture; the back stack lives in one NavController; deep links declared on
  destinations. Predictive back works through it (mandatory polish at targetSdk 35/36).
- Legacy fragment-based XML nav graphs: keep during transition; replace screen-by-screen as
  features are rebuilt in Compose. Do not maintain two parallel full graphs longer than one
  roadmap milestone.

### Build — Version catalogs, convention plugins, KSP

- `gradle/libs.versions.toml` is the single source of dependency truth — audits read it first
  (`02-engineering/03-dependency-analysis.md`).
- Multi-module apps get **convention plugins** (in `build-logic/`) instead of `subprojects {}`
  blocks or copy-pasted `build.gradle.kts` config; single-module apps skip this — do not
  over-engineer.
- **KSP, never kapt**: kapt generates Java stubs (slow) and is in maintenance mode; Room, Hilt,
  and Glide-replacements all support KSP. Running both processors causes its own pitfall class —
  see `08-knowledge/android/common-pitfalls.md`.

---

## Adoption Order for Legacy Apps (leverage-first)

Sequence migrations so each step makes later steps cheaper. Standard order for P4 roadmaps
(`01-core/prompts/04-roadmap-generation.md`):

1. **Toolchain floor**: Gradle wrapper → AGP → Kotlin 2.1+/K2 → JDK 17 toolchain → version
   catalog. Everything else depends on this; do it first, in small green-build commits.
2. **kapt → KSP** for existing processors. Build-speed win, removes the dual-processor trap.
3. **Coroutines at the seams**: convert async entry points (network calls, DB access) to suspend
   functions/Flow. This unblocks Room/DataStore/Compose, which all assume coroutines.
4. **SharedPreferences → DataStore** and **SQLite → Room** (data layer stabilizes before UI work).
5. **Hilt** skeleton (Application, ViewModels) so new Compose screens get clean injection.
6. **Compose screen-by-screen**, starting with the simplest high-traffic screen; M3 theme first
   (`03-design/05-material3-modernization.md`).
7. **Navigation consolidation** once a majority of screens are Compose.
8. Networking serializer swap (Gson → kotlinx.serialization) whenever the API layer is touched.

## Do-Not-Adopt List

| Avoid | Reason |
|---|---|
| Single-maintainer libraries on core paths (storage, networking, billing, crypto) | Bus factor 1; abandonment strands the app — prefer AndroidX/major-vendor equivalents even if less ergonomic |
| Alpha/beta artifacts in production builds | API churn and unfixed crashers; alphas are for spikes on branches — stable or RC only on `main` |
| Snapshot builds (`-SNAPSHOT`) | Non-reproducible builds; a CI run tomorrow compiles different code than today |
| New abstractions over Compose/Room "to stay flexible" | YAGNI wrapper layers double migration cost next cycle |
| Cross-platform UI rewrites (Flutter/RN/KMP-UI) of working native apps | Out of factory scope; modernize in place — a rewrite is a business decision, not a modernization task |

## Method docs that operationalize this stack

This doc names *what* to build with; the numbered method docs say *how* to get a target app there
and how to prove it. Cite these from P4 roadmaps and the engineering checklists:

| Concern | Method doc |
|---|---|
| Test coverage for the migrated layers (Room DAOs, ViewModels, Compose, migrations) | `02-engineering/08-testing-strategy.md` |
| R8 / resource shrinking — keep rules, full-mode safety, size budget (pairs with pitfall B1) | `02-engineering/09-r8-shrinking.md` |
| Performance audit — startup, jank, baseline profiles; thresholds in `08-knowledge/android/play-vitals-performance.md` | `02-engineering/10-performance-audit.md` |
| Room/DataStore migration safety (pairs with pitfall B5) | `02-engineering/11-data-migration-safety.md` |
| CI to enforce the above continuously | `02-engineering/12-ci-setup.md` |

Every audit finding that cites this doc should link the relevant layer section, severity-rated per
the scales in `01-core/CONVENTIONS.md`.
