# Engineering System — Map & Operating Rules

This directory is the **method library** for all engineering work the factory performs on a target
Android app: it defines *how* to discover, audit, inventory, migrate, and verify. The prompts in
`01-core/prompts/` define *when* each method runs and what artifacts it must produce; every
engineering prompt delegates its technique to a file here. A Claude agent executing any P1–P4 or P8
phase should open the relevant method doc below **before** touching the target repo.

## The Golden Rule

1. **Audit before touching.** Never modify a build file, manifest, or dependency until the
   relevant inventory/audit method (`01`–`06`) has been run and its output recorded in
   `docs/factory/audits/` of the target repo.
2. **Verify after touching.** Every change phase ends by passing the verification gate in
   `02-engineering/07-build-verification.md` at the tier mandated for that change type.
3. **One migration at a time, in dependency order.** Gradle → AGP → Kotlin → SDK → libraries.
   Never combine two migrations in one commit; never start the next until the previous one passes
   its verification tier and `docs/factory/PROJECT_STATE.md` is updated.

## Method Docs

| Doc | Method | Consumed by lifecycle phase | Consuming prompts |
|---|---|---|---|
| `02-engineering/01-discovery.md` | Read a Gradle project in canonical order; extract build facts; red-flag scan | P1 Discovery | `01-core/prompts/01-discovery.md` |
| `02-engineering/02-architecture-mapping.md` | Identify architecture pattern, layers, DI, navigation, data flow; score health 1–5 | P1, P3 | `01-discovery.md`, `03-modernization-audit.md` |
| `02-engineering/03-dependency-analysis.md` | Dump dependency graph; classify each dependency; map deprecated → replacement | P3 | `03-modernization-audit.md`, `04-roadmap-generation.md` |
| `02-engineering/04-sdk-inventory.md` | Detect third-party SDKs (ads/analytics/crash/billing); record privacy impact | P1, P3, P8 | `01-discovery.md`, `03-modernization-audit.md`, `16-release-preparation.md` |
| `02-engineering/05-service-inventory.md` | Inventory manifest components and background work; map API 12–16 breakage | P3 | `03-modernization-audit.md`, `04-roadmap-generation.md` |
| `02-engineering/06-manifest-audit.md` | Permission, exported-component, network, and backup-rule audit on the merged manifest; **Step 0 owns the canonical merged-manifest path** | P3, P8 | `03-modernization-audit.md`, `15-final-verification.md` |
| `02-engineering/07-build-verification.md` | Tiered build/test/install gate (Tiers 1–3, canonical numbering) after any code change | P4, P6, P8 (after every change) | All implementation prompts; `15-final-verification.md` |
| `02-engineering/08-testing-strategy.md` | Assess test posture; define the test pyramid, coverage targets, and which tests guard each migration | P3, P4 | `03-modernization-audit.md`, `04-roadmap-generation.md` |
| `02-engineering/09-r8-shrinking.md` | Enable/repair R8 + resource shrinking; keep-rule authoring; mapping-file handling | P4, P8 | `04-roadmap-generation.md`, `15-final-verification.md`; gated by `07-build-verification.md` Tier 3 |
| `02-engineering/10-performance-audit.md` | Cold-start, jank, and memory measurement; SDK init-cost attribution; baseline profiles | P3, P8 | `03-modernization-audit.md`, `15-final-verification.md` |
| `02-engineering/11-data-migration-safety.md` | Protect user data across upgrades: Room/SQLite migrations, SharedPreferences→DataStore, upgrade-path install testing | P4, P8 | `04-roadmap-generation.md`, `16-release-preparation.md`; upgrade-path test runs inside `07-build-verification.md` Tier 3 |
| `02-engineering/12-ci-setup.md` | Stand up CI (build + lint + test on PR); cache strategy; populates the Tech Stack Snapshot CI row | P4 | `04-roadmap-generation.md` |

## Migration Guides

| Guide | Scope | Phase | Run order |
|---|---|---|---|
| `02-engineering/migrations/gradle-migration.md` | Gradle wrapper + build-script upgrades, version catalogs | P4 | 1st |
| `02-engineering/migrations/agp-migration.md` | Android Gradle Plugin upgrades, AGP↔Gradle↔JDK compatibility | P4 | 2nd |
| `02-engineering/migrations/kotlin-migration.md` | Kotlin/K2, kapt→KSP, synthetics removal, coroutines adoption | P4 | 3rd |
| `02-engineering/migrations/android-sdk-migration.md` | compileSdk/targetSdk raises and per-API behavior changes | P4 | 4th |

The dependency order exists because each layer constrains the next: the Gradle version caps which
AGP you can use, AGP caps Kotlin plugin compatibility, and Kotlin/AGP together cap which Jetpack
artifacts (and therefore which compileSdk) resolve cleanly. Reversing the order produces
unresolvable version conflicts mid-migration.

## How Methods and Prompts Interlock

```
01-core/prompts/03-modernization-audit.md   (the WHEN: phase P3, inputs, outputs)
        │ delegates technique to
        ▼
02-engineering/02..06 (the HOW: commands, file paths, decision tables)
        │ findings recorded as ENG-### in
        ▼
docs/factory/audits/ENGINEERING_AUDIT.md    (instantiated from 09-templates/engineering-audit-report.md)
        │ prioritized by 01-core/prompts/04-roadmap-generation.md into
        ▼
docs/factory/PROJECT_STATE.md → Roadmap     → executed in P4 via migrations/* → gated by 07-build-verification.md
```

## Canonical Paths (engineering artifacts in the target repo)

These paths are defined once — here and in `01-core/CONVENTIONS.md` — and every engineering doc
references them rather than inventing variants:

| Artifact | Canonical location | Owning method |
|---|---|---|
| Dependency graph dumps | `docs/factory/audits/deps-<config>.txt` (`<config>` = Gradle configuration name, e.g. `deps-releaseRuntimeClasspath.txt`) | `02-engineering/03-dependency-analysis.md` Step 1 |
| Merged manifest | **Never restated** — obtain it via `02-engineering/06-manifest-audit.md` Step 0, which owns both AGP-version path variants | `06-manifest-audit.md` |
| Build baselines (build time, AAB/APK size) | `docs/factory/PROJECT_STATE.md` → **Tech Stack Snapshot** | `02-engineering/07-build-verification.md` Baseline Capture Rule |
| Audit artifacts (DISCOVERY, ENGINEERING_AUDIT) | `docs/factory/audits/` | per the artifact-contract matrix in `01-core/CONVENTIONS.md` |

## Conventions Used Across All Engineering Docs

- **Finding IDs**: engineering findings are `ENG-NNN` per the ID registry in
  `01-core/CONVENTIONS.md`, recorded in `docs/factory/audits/ENGINEERING_AUDIT.md`.
- **Scales**: Severity, Effort, and Status use the canonical scales in `01-core/CONVENTIONS.md`
  (Critical/High/Medium/Low · S/M/L/XL · Not Started/In Progress/Blocked/Done/Deferred).
- **Tech baseline (2026-06-10)**: compileSdk = targetSdk = 36, minSdk 26 recommended, JDK 17,
  Kotlin 2.1+ with K2, KSP (never kapt), AGP 8.9+, Gradle 8.13+, version catalogs, Compose BOM
  2025.x + Material 3, Hilt, Room, DataStore, WorkManager, Coroutines/Flow. Always confirm the
  current latest stable versions before recommending — see `08-knowledge/android/version-matrix.md`.
- **Commands**: all `./gradlew` commands are written POSIX-style; on Windows PowerShell use
  `.\gradlew.bat` with identical arguments.
- **Secrets**: never print keystore passwords, API keys, or signing config values into any audit
  artifact. Record presence and location only (e.g., "signing config reads from `keystore.properties`,
  not committed").

## Outputs & Where They Feed

| Output | Produced by | Feeds into |
|---|---|---|
| `docs/factory/audits/DISCOVERY.md` | Methods 01, 02, 04 via prompt 01 | P2 project memory; P3 audit scoping |
| `docs/factory/audits/ENGINEERING_AUDIT.md` (ENG- findings) | Methods 02–06 via prompt 03 | P3 roadmap generation; `07-checklists/engineering-modernization.md` |
| Modernization roadmap (in `PROJECT_STATE.md`) | Prompt 04 from ENG- findings | P4 execution order |
| Verification records (build time, artifact size) | Method 07 | `PROJECT_STATE.md` Tech Stack Snapshot; `docs/factory/CHANGELOG.md` |
| Data-safety implications table | Method 04 | `08-knowledge/stores/data-safety-mapping.md`; `04-aso/workflows/data-safety-listing.md`; P8 release prep (`10-releases/release-workflow.md`) |
| Dependency dumps (`audits/deps-<config>.txt`) | Method 03 (re-run after each P4 migration) | Graph-drift diffing; method 04 seed list |

## When You Are Lost

- Don't know the project layout yet → start at `02-engineering/01-discovery.md`, step 1.
- Build broke after a change → `02-engineering/07-build-verification.md`, "Reading failures".
- Unsure whether a dependency is safe to keep → `02-engineering/03-dependency-analysis.md`
  classification table.
- Play Console flagged a permission or data-safety issue → `02-engineering/06-manifest-audit.md`
  and `08-knowledge/stores/policy-landmines.md`.
- App feels slow / cold start regressed → `02-engineering/10-performance-audit.md`.
- Release crashes that debug never showed → `02-engineering/09-r8-shrinking.md` keep-rule section.
- Worried an update will eat user data → `02-engineering/11-data-migration-safety.md`.
