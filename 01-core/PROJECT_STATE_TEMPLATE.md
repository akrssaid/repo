# PROJECT_STATE_TEMPLATE — Project Memory Template

This template is instantiated into each target Android app as `docs/factory/PROJECT_STATE.md` —
the single source of truth for that project's status, facts, decisions, and history. Every
Claude session reads the instance at bootstrap and updates it before ending (per
`01-core/CLAUDE_MASTER.md`). Everything between the `=== TEMPLATE START ===` and
`=== TEMPLATE END ===` markers is copied into the instance with every `{{SNAKE_CASE}}` token
replaced; the markers, the Field Guide, and the Usage section below are not copied.

=== TEMPLATE START ===

# PROJECT_STATE — {{APP_NAME}}

> Single source of truth for this project. Read at every session start; update before every
> session end. Maintained per `01-core/CLAUDE_MASTER.md` of the android-product-factory.

| | |
|---|---|
| App name | {{APP_NAME}} |
| Package ID | {{PACKAGE_ID}} |
| Factory version | {{FACTORY_VERSION}} |
| Onboarded | {{ONBOARDED_DATE}} |
| Target repo path | {{TARGET_REPO_PATH}} |
| Stores published | {{STORES_PUBLISHED}} |

## Current Status

| | |
|---|---|
| Current phase | {{CURRENT_PHASE}} |
| Status | {{CURRENT_STATUS}} |
| Next action | {{NEXT_ACTION}} |
| Blockers | {{BLOCKERS}} |

## Phase Tracker

Status values use the canonical Status scale in `01-core/CONVENTIONS.md`. Done only when the
phase gate in `01-core/PROJECT_LIFECYCLE.md` has passed. Artifact links are repo-relative.

| Phase | Name | Status | Date | Artifact |
|---|---|---|---|---|
| P1 | Discovery | {{P1_STATUS}} | {{P1_DATE}} | {{P1_ARTIFACT}} |
| P2 | Project Memory | {{P2_STATUS}} | {{P2_DATE}} | {{P2_ARTIFACT}} |
| P3 | Engineering Audit & Roadmap | {{P3_STATUS}} | {{P3_DATE}} | {{P3_ARTIFACT}} |
| P4 | Modernization | {{P4_STATUS}} | {{P4_DATE}} | {{P4_ARTIFACT}} |
| P5 | Design Audit & Handover | {{P5_STATUS}} | {{P5_DATE}} | {{P5_ARTIFACT}} |
| P6 | Redesign Implementation | {{P6_STATUS}} | {{P6_DATE}} | {{P6_ARTIFACT}} |
| P7 | ASO & Store Assets | {{P7_STATUS}} | {{P7_DATE}} | {{P7_ARTIFACT}} |
| P8 | Verification & Release | {{P8_STATUS}} | {{P8_DATE}} | {{P8_ARTIFACT}} |

## App Identity

| | |
|---|---|
| Category | {{APP_CATEGORY}} |
| Value proposition | {{VALUE_PROPOSITION}} |
| Target user | {{TARGET_USER}} |
| Monetization model | {{MONETIZATION_MODEL}} |

## Tech Stack Snapshot

As of {{STACK_SNAPSHOT_DATE}}. Update after every P4 session that changes any row.

| Item | Value |
|---|---|
| minSdk | {{MIN_SDK}} |
| targetSdk | {{TARGET_SDK}} |
| compileSdk | {{COMPILE_SDK}} |
| Kotlin | {{KOTLIN_VERSION}} |
| AGP | {{AGP_VERSION}} |
| Gradle | {{GRADLE_VERSION}} |
| JDK toolchain | {{JDK_VERSION}} |
| UI toolkit | {{UI_TOOLKIT}} |
| DI | {{DI_FRAMEWORK}} |
| CI | {{CI_SETUP}} |
| Key libraries | {{KEY_LIBRARIES}} |

## Architecture Summary

{{ARCHITECTURE_SUMMARY}}

## Module Map

| Module | Type | Purpose | Depends on |
|---|---|---|---|
| {{MODULE_NAME}} | {{MODULE_TYPE}} | {{MODULE_PURPOSE}} | {{MODULE_DEPENDENCIES}} |

## Store Presence

| Store | Listing URL | Rating | Installs band |
|---|---|---|---|
| {{STORE_NAME}} | {{STORE_LISTING_URL}} | {{STORE_RATING}} | {{STORE_INSTALLS_BAND}} |

## Known Risks

Full register: {{RISK_REGISTER_LINK}} (instantiated from `09-templates/risk-register.md`).
Top risks summarized here; keep in sync with the register.

| Risk | Severity | Mitigation / status |
|---|---|---|
| {{RISK_SUMMARY}} | {{RISK_SEVERITY}} | {{RISK_MITIGATION}} |

## Open Questions

Questions awaiting an operator answer or missing evidence. Review at every session bootstrap.
A resolution becomes a Decision Log row or a fact-section update; flip the question's status to
Resolved with a pointer — do not delete the row.

| # | Question | Raised (date, phase) | Blocking? | Status |
|---|---|---|---|---|
| 1 | {{OPEN_QUESTION}} | {{QUESTION_RAISED}} | {{QUESTION_BLOCKING}} | {{QUESTION_STATUS}} |

## Decision Log

Record every operator decision, scope change, escalation outcome, and rule override. Append-only.

| Date | Decision | Rationale | Made by |
|---|---|---|---|
| {{DECISION_DATE}} | {{DECISION}} | {{DECISION_RATIONALE}} | {{DECISION_MAKER}} |

## Backlog

Prioritized top-down. Severity/Effort/Status use the canonical scales in
`01-core/CONVENTIONS.md`. Findings deferred from any phase land here; P3 owns re-triage.

| # | Item | Severity | Effort | Source phase | Status |
|---|---|---|---|---|---|
| 1 | {{BACKLOG_ITEM}} | {{BACKLOG_SEVERITY}} | {{BACKLOG_EFFORT}} | {{BACKLOG_SOURCE_PHASE}} | {{BACKLOG_STATUS}} |

## Artifact Index

One row per factory artifact produced for this app; canonical paths and names per
`01-core/CONVENTIONS.md`. Update whenever an artifact is created, revised (`_v<N>` bump), or
superseded.

| Artifact | Path | Version | Date | Status |
|---|---|---|---|---|
| {{ARTIFACT_NAME}} | {{ARTIFACT_PATH}} | {{ARTIFACT_VERSION}} | {{ARTIFACT_DATE}} | {{ARTIFACT_STATUS}} |

## Release History

One row per app release shipped during factory engagements. Append-only.

| versionName | versionCode | Date | Rollout outcome | Report |
|---|---|---|---|---|
| {{RELEASE_VERSION_NAME}} | {{RELEASE_VERSION_CODE}} | {{RELEASE_DATE}} | {{ROLLOUT_OUTCOME}} | {{RELEASE_REPORT_LINK}} |

## Session Log

Append-only; newest row last. One row per session, written during the Session End Procedure.
Detail lives in `docs/factory/CHANGELOG.md`; this table is the index.

| Date | Phase | Summary | By |
|---|---|---|---|
| {{SESSION_DATE}} | {{SESSION_PHASE}} | {{SESSION_SUMMARY}} | {{SESSION_AGENT}} |

=== TEMPLATE END ===

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` | Human-readable app name | `Solar PDF Reader` |
| `{{PACKAGE_ID}}` | Application ID from the default config | `com.solarcamera.pdfreader` |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` at onboarding; compared against the factory at every bootstrap (CLAUDE_MASTER step 3a) | `1.1.0` |
| `{{ONBOARDED_DATE}}` | Date P2 instantiated this file (absolute) | `2026-06-10` |
| `{{TARGET_REPO_PATH}}` | Absolute path to the target repo | `C:\Users\ilkar\...\solar-pdf-reader` |
| `{{STORES_PUBLISHED}}` | Comma list of stores the app is live on, or `None yet` | `Google Play, RuStore` |
| `{{CURRENT_PHASE}}` | Active phase, `PX — Name` | `P3 — Engineering Audit & Roadmap` |
| `{{CURRENT_STATUS}}` | Status of the current phase (canonical scale) | `In Progress` |
| `{{NEXT_ACTION}}` | One imperative sentence the next session executes first | `Run dependency analysis on :core module` |
| `{{BLOCKERS}}` | Named blockers, or `None` | `Awaiting operator keystore decision` |
| `{{P1_STATUS}}`…`{{P8_STATUS}}` | Per-phase status (canonical scale) | `Done` |
| `{{P1_DATE}}`…`{{P8_DATE}}` | Date the status last changed, or `—` | `2026-06-12` |
| `{{P1_ARTIFACT}}`…`{{P8_ARTIFACT}}` | Repo-relative link to the phase's exit artifact, or `—` | `docs/factory/audits/DISCOVERY.md` |
| `{{APP_CATEGORY}}` | Store category + playbook match if any | `Tools / document-reader (see 06-playbooks/document-reader.md)` |
| `{{VALUE_PROPOSITION}}` | One sentence: what the app does for whom | `Fast offline PDF reading with annotation` |
| `{{TARGET_USER}}` | Primary user profile | `Students and office workers on mid-range devices` |
| `{{MONETIZATION_MODEL}}` | How the app earns | `AdMob banners + remove-ads IAP` |
| `{{STACK_SNAPSHOT_DATE}}` | Date the snapshot was last verified | `2026-06-10` |
| `{{MIN_SDK}}` / `{{TARGET_SDK}}` / `{{COMPILE_SDK}}` | SDK levels from build config | `26` / `36` / `36` |
| `{{KOTLIN_VERSION}}` | Kotlin plugin version (or `None — Java only`) | `2.1.20` |
| `{{AGP_VERSION}}` | Android Gradle Plugin version | `8.9.1` |
| `{{GRADLE_VERSION}}` | Gradle wrapper version | `8.13` |
| `{{JDK_VERSION}}` | JDK toolchain version | `17` |
| `{{UI_TOOLKIT}}` | UI technology and mix | `Compose (BOM 2025.05) + 3 legacy XML screens` |
| `{{DI_FRAMEWORK}}` | DI in use, or `None (manual)` | `Hilt 2.55` |
| `{{CI_SETUP}}` | CI system and config location, or `None` | `GitHub Actions (.github/workflows/android.yml)` |
| `{{KEY_LIBRARIES}}` | Notable libraries with versions, comma list | `Room 2.7, Retrofit 2.11, Coil 3.1` |
| `{{ARCHITECTURE_SUMMARY}}` | 3–8 sentence prose summary of architecture, layering, navigation, data flow | `Single-activity MVVM; …` |
| `{{MODULE_NAME}}` / `{{MODULE_TYPE}}` / `{{MODULE_PURPOSE}}` / `{{MODULE_DEPENDENCIES}}` | One row per Gradle module (duplicate the row) | `:core` / `android-library` / `shared models, db` / `—` |
| `{{STORE_NAME}}` / `{{STORE_LISTING_URL}}` / `{{STORE_RATING}}` / `{{STORE_INSTALLS_BAND}}` | One row per store (duplicate the row); rating/installs from the live listing or `(estimate)` / `Unpublished` | `Google Play` / `https://play.google.com/store/apps/details?id=…` / `4.2` / `100K–500K` |
| `{{RISK_REGISTER_LINK}}` | Repo-relative path to the risk register instance | `docs/factory/reports/RISK_REGISTER.md` |
| `{{RISK_SUMMARY}}` / `{{RISK_SEVERITY}}` / `{{RISK_MITIGATION}}` | One row per top risk (duplicate the row) | `Native lib not 16KB-page safe` / `High` / `Upgrade in P4, step 5` |
| `{{OPEN_QUESTION}}` | One row per unresolved question (append rows) | `Is the legacy widget still used by real users?` |
| `{{QUESTION_RAISED}}` | Date and phase the question surfaced | `2026-06-12, P3` |
| `{{QUESTION_BLOCKING}}` | `Yes — <what it blocks>` or `No` | `Yes — P4 step 6` |
| `{{QUESTION_STATUS}}` | `Open` or `Resolved — see <Decision Log row / section>` | `Open` |
| `{{DECISION_DATE}}` / `{{DECISION}}` / `{{DECISION_RATIONALE}}` / `{{DECISION_MAKER}}` | One row per decision (append rows) | `2026-06-10` / `Skip RuStore for v2` / `No RU market focus` / `Operator` |
| `{{BACKLOG_ITEM}}` / `{{BACKLOG_SEVERITY}}` / `{{BACKLOG_EFFORT}}` / `{{BACKLOG_SOURCE_PHASE}}` / `{{BACKLOG_STATUS}}` | One row per backlog item (append rows, keep priority order) | `Replace AsyncTask in ExportService` / `High` / `M` / `P3` / `Not Started` |
| `{{ARTIFACT_NAME}}` | Artifact name as in the `01-core/CONVENTIONS.md` artifact matrix | `DESIGN_HANDOVER` |
| `{{ARTIFACT_PATH}}` | Repo-relative path to the current version | `docs/factory/handovers/DESIGN_HANDOVER_v2.md` |
| `{{ARTIFACT_VERSION}}` | `v<N>` for versioned artifacts, `—` otherwise | `v2` |
| `{{ARTIFACT_DATE}}` | Date of last create/revise | `2026-06-17` |
| `{{ARTIFACT_STATUS}}` | Lifecycle status (handovers: Draft / Delivered / Accepted / Rejected; others: canonical Status scale) | `Accepted` |
| `{{RELEASE_VERSION_NAME}}` / `{{RELEASE_VERSION_CODE}}` | versionName / versionCode actually shipped | `2.4.0` / `240` |
| `{{RELEASE_DATE}}` | Date the release went live (first stage) | `2026-07-01` |
| `{{ROLLOUT_OUTCOME}}` | How the staged rollout ended | `100% on 2026-07-05; crash-free 99.6%` or `Halted at 25% — ANR spike, rolled back` |
| `{{RELEASE_REPORT_LINK}}` | Repo-relative link to the release report | `docs/factory/reports/RELEASE_REPORT_v2.4.0.md` |
| `{{SESSION_DATE}}` / `{{SESSION_PHASE}}` / `{{SESSION_SUMMARY}}` / `{{SESSION_AGENT}}` | One row per session (append rows) | `2026-06-10` / `P2` / `Instantiated project memory` / `Claude (Principal Android Engineer)` |

## Usage

1. **Who instantiates it**: Claude, acting as Principal Android Engineer, executing
   `01-core/prompts/02-project-memory.md` during phase P2.
2. **When**: immediately after P1 Discovery passes its gate — this file's facts come from
   `docs/factory/audits/DISCOVERY.md` plus recorded operator answers, never from guesses. Any
   token whose value is unknown sends you back to P1 (see `01-core/PROJECT_LIFECYCLE.md`, P2
   loop-back rules).
3. **Where the instance lives**: `docs/factory/PROJECT_STATE.md` in the TARGET app repo. One
   instance per app; the factory copy is never edited per-project.
4. **Instantiation mechanics**: copy everything between the TEMPLATE START/END markers (without
   the markers); replace every token; duplicate the single example rows of Module Map, Store
   Presence, Known Risks, Open Questions, Decision Log, Backlog, Artifact Index, Release
   History, and Session Log as needed (at least one real row each — use `—` cells for genuinely
   empty tables, never leave a token). The P2 gate fails if any `{{...}}` remains.
5. **Section names are normative.** Prompts, checklists, and schemas reference these section
   headings verbatim (see `01-core/CONVENTIONS.md`). Never rename, merge, or reorder sections
   in an instance — a consumer that cannot find "Phase Tracker" by name is broken by you.
6. **Update cadence**: every session updates Current Status, the relevant Phase Tracker row, and
   appends a Session Log row before ending (CLAUDE_MASTER, Sections 2a and 3). Beyond that,
   each prompt's Documentation Update Rules name the exact sections it must update — follow
   them. Typical mapping: Tech Stack Snapshot after P4 work; Store Presence and Release History
   after P7/P8 work; Artifact Index whenever an artifact is created or revised; Open Questions
   and Decision Log whenever questions are raised or decided; Backlog whenever findings are
   deferred.
7. **Append-only sections**: Decision Log, Release History, and Session Log rows are never
   edited or deleted — corrections are new rows. Other sections are living state and are edited
   in place.
