# Framework Changelog

This is the changelog of the **android-product-factory repository itself** — not of any target
app (target apps keep their own `docs/factory/CHANGELOG.md`, instantiated from
`01-core/CHANGELOG_TEMPLATE.md`). Format follows Keep a Changelog: newest entry first, one
`## [version] — date` heading per release, changes grouped under `### Added` / `### Changed` /
`### Fixed` / `### Removed` / `### Deprecated`. The factory's version number itself is governed
by `VERSION.md`.

## [Unreleased]

<!--
  Standing instruction: every change to the factory lands here, in the same commit that makes
  the change. One bullet per change, under the appropriate ### heading (Added / Changed /
  Fixed / Removed / Deprecated), each bullet naming the affected file by repo-relative path
  and saying what changed and why in one line. When a factory version is cut per VERSION.md,
  this section's contents move under a new ## [X.Y.Z] — YYYY-MM-DD heading and this section
  returns to empty (comment retained).
-->

_No unreleased changes._

## [1.1.0] — 2026-06-10

The four-persona principal review of v1.0 driving a contracts-first refactor: every contract
(paths, names, IDs, thresholds) is now defined ONCE and cited everywhere, so a zero-context agent
can execute any phase without guessing. New methods extend the lifecycle past launch (post-launch
monitoring, experiments, ratings/reviews) and deepen engineering and design coverage. Factory
version governed by `VERSION.md`; this release is tagged `factory/v1.1.0`.

### Added

- **Root & core docs** — `QUICK_REFERENCE.md` (root), `01-core/CONVENTIONS.md` (single source of
  truth for contracts), `01-core/README.md`, `01-core/WORKED_EXAMPLE.md` (4 docs).
- **Engineering methods** (`02-engineering/`) — `08-testing-strategy.md`, `09-r8-shrinking.md`,
  `10-performance-audit.md`, `11-data-migration-safety.md`, `12-ci-setup.md` (5 methods).
- **Design methods** (`03-design/`) — `10-motion-design.md`, `11-adaptive-layout.md`,
  `12-design-qa.md` (3 methods).
- **ASO workflows** (`04-aso/workflows/`) — `ratings-reviews.md`, `experiment-program.md`,
  `post-launch-monitoring.md`, `data-safety-listing.md` (4 workflows).
- **Handover schema** (`05-handover/`) — `REDESIGN_PROPOSAL_SCHEMA.md` for the Claude Design
  redesign-proposal artifact (prompt 07).
- **Checklist** (`07-checklists/`) — `aso-post-release.md`, the post-release gate.
- **Knowledge docs** (`08-knowledge/`) — `android/play-vitals-performance.md` (owns the 1.09%/0.47%
  bad-behavior thresholds and startup bands), `design/ux-writing-essentials.md`,
  `stores/data-safety-mapping.md` (3 docs).
- **Templates** (`09-templates/`) — `experiment-log.md`, `verification-report.md` (2 templates).
- **Category playbooks** (`06-playbooks/`) — `pdf-scanner.md`, `qr-barcode-scanner.md`
  (2 playbooks).

### Changed

- **`01-core/CONVENTIONS.md` is now the single source of truth** — ID registry, severity/effort/
  status scales, branch/commit/tag formats, canonical `docs/factory/` paths, and the
  artifact-contract matrix live here; every other file cites it instead of restating it.
- **Reconciled contract drift** across prompts, templates, schemas, and checklists so artifact
  names, paths, and gate references match end to end.
- **Motion pipeline added end-to-end** — extraction census → DESIGN handover inventory → motion
  vocabulary in `material3-essentials.md` + `03-design/10-motion-design.md` → REDESIGN proposal
  Motion lines → IMPLEMENTATION schema Motion Specification + ticket `Motion:` field →
  design-modernization checklist item.
- **COPY-ticket flow replaces the blanket string-freeze** — prompt 06 declares frozen vs
  redesignable strings, 07 proposes copy, 08 emits `COPY-NNN` tickets, 09 implements them;
  un-ticketed string changes remain banned.
- **ASO lifecycle extended past launch** — ratings/reviews, an experiment program, and a
  post-launch monitoring loop now follow store go-live.
- **Rollout gates single-sourced** — `10-releases/release-workflow.md` stage 4 owns all stage/
  dwell/threshold numbers (10→25→50→100%, 24h dwell, HOLD at crash +0.2pp / ANR ≥0.40%); prompt
  16 and the RELEASE schema defer to it.
- **`PROJECT_STATE_TEMPLATE.md`** gained the Open Questions, Artifact Index, and Release History
  sections (plus a CI row in the tech-stack snapshot).
- **CLAUDE_MASTER bootstrap** now compares the recorded factory version against `VERSION.md` and
  runs the upgrade procedure on a major mismatch before any phase work.

### Fixed

- Prompt 02 token mismatch (canonical `{{PACKAGE_ID}}`).
- P1 build-policy contradiction reconciled (read-only Gradle + one documented `assembleDebug`
  attempt across prompt 01, `02-engineering/01-discovery.md`, and the CLAUDE_MASTER P1 gate).
- Broken shell commands corrected, including the versioning-policy merged-manifest verification
  (`:app:processDebugMainManifest`, never `:app:dependencies`).
- Malformed greps in `02-engineering/02-architecture-mapping.md` (no duplicate `-r`, no `-l`+`-n`).
- ASO schema example character-count errors made actually correct.
- Merged-manifest path drift collapsed to a single statement in `06-manifest-audit.md` Step 0.
- Verdict vocabulary normalized to PASS / PASS with waivers / FAIL.
- Git tag formats normalized (`factory/vX.Y.Z`, `release/vX.Y.Z`).

### Removed

- The `A11Y-DES-xxx` ID format (accessibility is its own `A11Y-001` stream).
- The "CONDITIONAL" verification/handover verdict.
- The "Partial" ASO handover status.
- Duplicated convention definitions scattered across files (now centralized in CONVENTIONS.md).
- Abolished asset paths (`assets/screenshots/store/`, `assets/feature-graphic/`, `assets/icons/`,
  `reports/listings/`).

## [1.0.0] — 2026-06-10

Initial release of the factory: the complete operating system for modernizing Android apps
end-to-end (discovery → project memory → engineering audit → modernization → design audit →
redesign → ASO → release), built for use with Claude Code and Claude Design.

### Added

- **Root docs** — `README.md`, `HOW_TO_USE.md`, `BEST_PRACTICES.md`, `VERSION.md` (4 docs).
- **Core system** (`01-core/`) — `CLAUDE_MASTER.md` master instructions,
  `PROJECT_LIFECYCLE.md` defining phases P1–P8, plus the per-project state and changelog
  templates `PROJECT_STATE_TEMPLATE.md` and `CHANGELOG_TEMPLATE.md` (4 docs).
- **Prompt library** (`01-core/prompts/`) — 16 lifecycle prompts, `01-discovery.md` through
  `16-release-preparation.md`, each with the standard 9-section structure (Role → Failure
  Modes & Recovery), covering all eight lifecycle phases.
- **Engineering system** (`02-engineering/`) — 7 method docs (discovery, architecture mapping,
  dependency analysis, SDK inventory, service inventory, manifest audit, build verification)
  plus 4 migration guides (Android SDK, Kotlin, Gradle, AGP) under `migrations/`.
- **Design system** (`03-design/`) — 9 method docs: design audit, screen inventory, screenshot
  capture, design-system extraction, Material 3 modernization, icon redesign, splash redesign,
  component library, accessibility review.
- **ASO system** (`04-aso/`) — 3 store guides (Google Play, Huawei AppGallery, RuStore) and
  5 workflows (keyword research, competitor analysis, screenshot optimization, icon
  optimization, localization).
- **Handover schemas** (`05-handover/`) — 4 schemas: DESIGN, IMPLEMENTATION, ASO, and RELEASE
  handover, with the shared naming convention `<TYPE>_HANDOVER_v<N>.md`.
- **Category playbooks** (`06-playbooks/`) — 4 playbooks: document-reader, video-downloader,
  file-manager, data-recovery.
- **Checklists** (`07-checklists/`) — 7 checklist documents: engineering modernization, design
  modernization, accessibility, ASO launch, release readiness, handover validation, plus the
  directory guide.
- **Knowledge base** (`08-knowledge/`) — 12 knowledge docs across 5 categories: android
  (version matrix, modern stack, common pitfalls), design (Material 3 essentials, mobile UX
  principles, app icon principles), aso (ranking factors, conversion optimization), stores
  (policy landmines, store comparison), monetization (ads patterns, subscription patterns).
- **Report templates** (`09-templates/`) — 8 templates: engineering/design/ASO audit reports,
  screen inventory, keyword map, store listing, release report, risk register — each with a
  Field Guide and Usage section.
- **Release system** (`10-releases/`) — release workflow for target apps, app versioning
  policy (`M NN PP BB` versionCode standard, same-code multi-store strategy), and this
  framework changelog.

## Changelog update rules

1. **Same-commit rule.** Any commit that changes a factory file also updates the `[Unreleased]`
   section of this changelog. A factory change without a changelog bullet is an incomplete
   change; reviewers reject it.
2. **One bullet per change**, under the correct heading, citing the repo-relative path of every
   file touched, e.g. `- **Changed** 02-engineering/migrations/agp-migration.md — updated for
   AGP 8.10 namespace defaults.` Group multi-file changes of one logical edit into one bullet.
3. **Releasing a factory version.** Version semantics, bump criteria, and the release procedure
   for the factory itself are defined in `VERSION.md`. When cutting version `X.Y.Z`: move the
   `[Unreleased]` contents under a new `## [X.Y.Z] — YYYY-MM-DD` heading (absolute date),
   restore the empty `[Unreleased]` section with its instruction comment, update `VERSION.md`,
   and tag the commit `factory/vX.Y.Z`.
4. **Newest first, never rewrite history.** Published entries are immutable; corrections get a
   new entry under `### Fixed` referencing the erroneous one.
5. **Scope guard.** Target-app changes never appear here — they belong in that app's
   `docs/factory/CHANGELOG.md`. Only changes to files in this repository qualify.
