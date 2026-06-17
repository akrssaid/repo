# Architecture Mapping Method

This method turns a discovered codebase (per `02-engineering/01-discovery.md`) into an explicit
architecture map: the pattern in use, the layers and how cleanly they separate, the DI and
navigation mechanisms, the data-flow primitives, and the threading model. Its outputs — an ASCII
module/layer diagram, a component inventory table, and a 1–5 health score per dimension — feed
directly into `docs/factory/audits/ENGINEERING_AUDIT.md` (phase P3, via
`01-core/prompts/03-modernization-audit.md`) and shape the modernization roadmap. Map what *is*,
not what the README claims; code wins over documentation every time.

## Step 1 — Identify the Architectural Pattern

Run these counts first; they decide which pattern you are looking at:

```bash
grep -rl --include="*.kt" --include="*.java" "ViewModel" app/src/main | wc -l
grep -rl --include="*.kt" --include="*.java" "extends AppCompatActivity\|: AppCompatActivity\|: ComponentActivity" app/src/main
grep -rl --include="*.kt" --include="*.java" "Presenter" app/src/main | wc -l
grep -rl --include="*.kt" "UseCase\|Interactor" app/src/main | wc -l
grep -rl --include="*.kt" --include="*.java" "Repository" app/src/main | wc -l
grep -rl --include="*.kt" "sealed class.*Intent\|sealed interface.*Intent\|sealed class.*Event\|UiState" app/src/main | wc -l
```

```powershell
$src = Get-ChildItem -Recurse app\src\main -Include *.kt,*.java
($src | Select-String -List "ViewModel").Count
$src | Select-String -List "extends AppCompatActivity|: AppCompatActivity|: ComponentActivity" | Select-Object Path
($src | Select-String -List "Presenter").Count
($src | Where-Object Extension -eq ".kt" | Select-String -List "UseCase|Interactor").Count
($src | Select-String -List "Repository").Count
($src | Where-Object Extension -eq ".kt" | Select-String -List "sealed class.*Intent|sealed interface.*Intent|sealed class.*Event|UiState").Count
```

| Evidence | Verdict |
|---|---|
| ViewModels + Repositories + observable state (Flow/LiveData) | MVVM |
| MVVM signals **plus** sealed `Intent`/`Event` in, single immutable `UiState` out, a `reduce`/`Reducer` | MVI |
| `Presenter` classes holding `View` interfaces; Activity implements `XxxView` | MVP |
| Activities/Fragments > 500 lines doing network + DB + UI; few/no ViewModels | God-Activity / no architecture |
| Mixed (e.g., new screens MVVM, old screens God-Activity) | Record as "transitional"; map per-screen in the inventory table |

Check line counts of the largest UI classes — the single most honest architecture signal:

```bash
find app/src/main -name "*.kt" -o -name "*.java" | xargs wc -l | sort -rn | head -20
```

```powershell
Get-ChildItem -Recurse app\src\main -Include *.kt,*.java |
  Select-Object FullName, @{n='Lines'; e={(Get-Content $_.FullName | Measure-Object -Line).Lines}} |
  Sort-Object Lines -Descending | Select-Object -First 20
```

Any Activity/Fragment over ~800 lines is a finding regardless of declared pattern.

## Step 2 — Layer Detection

Heuristics, in order of reliability:

1. **Module names**: `:domain`, `:data`, `:core:network`, `:feature:*` — module boundaries are
   enforced layers (Gradle won't compile violations). Strongest separation evidence.
2. **Package names**: `ui`/`presentation`, `domain`, `data`/`repository`, `di`, `model`. Packages
   are *conventions only* — verify with import checks below.
3. **Class-name suffixes**: `*ViewModel`, `*UseCase`/`*Interactor`, `*Repository`, `*DataSource`,
   `*Dao`, `*ApiService`, `*Mapper`.

Verify the separation actually holds (convention violations are findings):

```bash
# UI classes importing data-layer internals directly = layering violation
grep -rn --include="*.kt" "import .*\.data\." app/src/main | grep "/ui/\|/presentation/" | head -30
# Domain layer importing Android framework = impure domain
grep -rn --include="*.kt" "import android\." . | grep "/domain/" | head -30
# Repositories called directly from Activities/Fragments (bypassing ViewModel)
grep -rn --include="*Activity.kt" --include="*Fragment.kt" "Repository(" app/src/main
```

```powershell
# UI classes importing data-layer internals directly = layering violation
Get-ChildItem -Recurse app\src\main -Filter *.kt | Where-Object FullName -match '\\(ui|presentation)\\' |
  Select-String "import .*\.data\." | Select-Object -First 30
# Domain layer importing Android framework = impure domain
Get-ChildItem -Recurse . -Filter *.kt | Where-Object FullName -match '\\domain\\' |
  Select-String "import android\." | Select-Object -First 30
# Repositories called directly from Activities/Fragments (bypassing ViewModel)
Get-ChildItem -Recurse app\src\main -Include *Activity.kt,*Fragment.kt | Select-String "Repository\("
```

## Step 3 — DI Detection

| Framework | Detection | Notes |
|---|---|---|
| Hilt | `@HiltAndroidApp` on Application; `dagger.hilt` deps; `@AndroidEntryPoint` count | Baseline choice; record coverage (% of entry points annotated) |
| Dagger 2 (plain) | `@Component` interfaces, `Dagger*Component` generated refs, no hilt artifacts | Migration to Hilt is usually an M/L roadmap item |
| Koin | `startKoin {}` in Application; `io.insert-koin` deps; `by inject()` / `koinViewModel()` | Acceptable; note kapt-free |
| Manual / Service Locator | A singleton `Injection`/`ServiceLocator`/`AppContainer` object; constructors `new`-ed in Activities | Record as "manual"; testability dimension capped at 3 |
| None | Direct instantiation + statics/singletons everywhere | Testability dimension capped at 2 |

```bash
grep -rn --include="*.kt" --include="*.java" "@HiltAndroidApp\|startKoin\|DaggerApp" app/src/main
grep -rc --include="*.kt" "@AndroidEntryPoint" app/src/main | grep -v ':0'
```

```powershell
Get-ChildItem -Recurse app\src\main -Include *.kt,*.java | Select-String "@HiltAndroidApp|startKoin|DaggerApp"
Get-ChildItem -Recurse app\src\main -Filter *.kt | Select-String "@AndroidEntryPoint" |
  Group-Object Path | Select-Object Name, Count
```

## Step 4 — Navigation

| Approach | Detection |
|---|---|
| Compose Navigation | `androidx.navigation:navigation-compose` dep; `NavHost(` composables; route constants or type-safe `@Serializable` routes |
| Navigation component (XML) | `res/navigation/*.xml` graphs; `findNavController()` |
| Manual intents | `startActivity(Intent(` scattered in UI classes; count them — bash: `grep -rc --include="*.kt" "startActivity(" app/src/main \| grep -v ':0'`; powershell: `(Get-ChildItem -Recurse app\src\main -Filter *.kt \| Select-String "startActivity\(").Count` |
| Fragment transactions (manual) | `supportFragmentManager.beginTransaction()` |
| Single-activity? | Count `<activity>` entries in merged manifest vs. screens. 1 activity + many screens = single-activity (modern); 1 activity per screen = legacy shape |

Record: how a new screen gets wired in, whether back-stack/deep-link handling is centralized, and
whether navigation arguments are type-safe.

## Step 5 — Data Flow & Threading

```bash
grep -rl --include="*.kt" "kotlinx.coroutines.flow" app/src | wc -l
grep -rl --include="*.kt" --include="*.java" "LiveData\|MutableLiveData" app/src | wc -l
grep -rl --include="*.kt" --include="*.java" "io.reactivex" app/src | wc -l   # RxJava 2/3
grep -rl --include="*.java" "rx\.\(Observable\|Single\)" app/src | wc -l      # RxJava 1 — abandoned, escalate
grep -rl --include="*.kt" --include="*.java" "AsyncTask\|new Thread(\|Handler()" app/src
grep -rn --include="*.kt" "runBlocking" app/src/main | grep -v test           # main-thread blocking risk
grep -rn --include="*.kt" "Dispatchers.Main\|Dispatchers.IO" app/src/main | wc -l
```

```powershell
$src = Get-ChildItem -Recurse app\src -Include *.kt,*.java
($src | Where-Object Extension -eq ".kt" | Select-String -List "kotlinx.coroutines.flow").Count
($src | Select-String -List "LiveData|MutableLiveData").Count
($src | Select-String -List "io.reactivex").Count                              # RxJava 2/3
($src | Where-Object Extension -eq ".java" | Select-String -List "rx\.(Observable|Single)").Count  # RxJava 1
$src | Select-String -List "AsyncTask|new Thread\(|Handler\(\)" | Select-Object Path
Get-ChildItem -Recurse app\src\main -Filter *.kt | Select-String "runBlocking" |
  Where-Object Path -notmatch "test"                                           # main-thread blocking risk
(Get-ChildItem -Recurse app\src\main -Filter *.kt | Select-String "Dispatchers.Main|Dispatchers.IO").Count
```

Record the **dominant** primitive and every **legacy** primitive still present. Mixed stacks
(RxJava + Flow, LiveData + StateFlow) are normal mid-migration but each legacy primitive becomes
an ENG- finding with a migration target (see `02-engineering/03-dependency-analysis.md`
replacement map). `AsyncTask` (deprecated since API 30 — still compiles and runs, which is exactly
why it lingers), bare `Thread`, and no-arg `Handler()` are automatic Medium+ findings.

## Step 6 — Produce the Diagram and Inventory

**Module/layer ASCII diagram** — modules as boxes, `implementation(project(...))` edges as arrows,
annotate each box with its layer and UI tech:

```
┌─────────────┐     ┌──────────────────┐
│ :app (UI,   │ ──▶ │ :feature:reader   │──┐
│  Compose)   │     │ (UI, Compose+XML) │  │   ┌──────────────┐
└─────┬───────┘     └──────────────────┘  ├──▶│ :core:data    │──▶ Room / Retrofit
      │      ┌──────────────────┐         │   │ (repositories)│
      └─────▶│ :core:ui (theme) │         │   └──────┬────────┘
             └──────────────────┘         │   ┌──────▼────────┐
                                          └──▶│ :core:domain  │ (pure Kotlin)
                                              └───────────────┘
```

Single-module apps still get a diagram — draw packages-as-layers instead and mark unenforced
boundaries with dashed arrows.

**Component inventory table** (one row per screen/feature):

| Screen/Feature | Entry (Activity/Fragment/Composable) | Pattern | ViewModel? | State primitive | Nav wiring | LoC (largest class) | Notes |
|---|---|---|---|---|---|---|---|

## Step 7 — Architecture Health Scoring Rubric

Score each dimension 1–5; the row anchors below are definitions, not vibes. Record the table plus
a one-line justification per score in `docs/factory/audits/ENGINEERING_AUDIT.md`.

| Dimension | 1 | 3 | 5 |
|---|---|---|---|
| Separation of concerns | God-Activities; UI does I/O directly | Layers by package convention; some violations found in Step 2 checks | Layers enforced by module boundaries; zero violations |
| Testability | No DI, statics/singletons; no unit tests possible without robolectric heroics | Constructor injection in places; ViewModels testable; <20% coverage of logic | Full DI; pure domain layer; fast JVM tests exist and run green |
| State management | Mutable fields on views; state lost on rotation | LiveData/StateFlow per screen but multiple sources of truth | Single immutable UiState per screen; process-death handled (SavedStateHandle) |
| Navigation | Manual intents + string extras everywhere | Centralized graph but untyped args, or mixed manual/graph | Single-activity, typed routes, deep links centralized |
| Dependency injection | None/manual `new` in UI | Partial DI (some entry points), or service locator | Hilt/Koin app-wide, scoped correctly, no field injection abuse outside entry points |

**Overall** = unweighted mean, rounded half down. Overall ≤ 2 ⇒ raise an ENG- finding proposing
"architecture stabilization before redesign" — phases P5–P6 are high-risk on a 1–2 codebase and
the roadmap (prompt `04-roadmap-generation.md`) must sequence stabilization first.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Pattern verdict + evidence | ENGINEERING_AUDIT.md "Architecture" section | Roadmap sequencing (P3→P4); risk register (`09-templates/risk-register.md`) |
| ASCII module/layer diagram | ENGINEERING_AUDIT.md + `PROJECT_STATE.md` Tech Stack Snapshot | Onboarding for every later prompt; P5 design handover context |
| Component inventory table | ENGINEERING_AUDIT.md | `03-design/02-screen-inventory.md` (P5) reuses screen list; P6 implementation planning |
| Health scores (5 dimensions + overall) | ENGINEERING_AUDIT.md scorecard | Go/no-go on redesign depth; ENG- finding generation |
| Layering-violation grep hits | ENG- findings with file:line | `07-checklists/engineering-modernization.md` items |
