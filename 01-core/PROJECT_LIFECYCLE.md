# PROJECT_LIFECYCLE — Phases P1–P8

This document is the authoritative definition of the android-product-factory lifecycle: eight
phases that take a target Android app from unknown codebase to modernized, redesigned, store-
optimized release. Every session operates inside exactly one phase (per
`01-core/CLAUDE_MASTER.md`, Section 7), and every phase has entry criteria, a quality gate, and
defined loop-backs. If a prompt, checklist, or session plan disagrees with this document about
what a phase is or when it is done, this document wins.

## Phase overview

| Phase | Name | Prompts used |
|---|---|---|
| P1 | Discovery | `01-core/prompts/01-discovery.md` |
| P2 | Project Memory | `01-core/prompts/02-project-memory.md` |
| P3 | Engineering Audit & Roadmap | `01-core/prompts/03-modernization-audit.md`, `01-core/prompts/04-roadmap-generation.md` |
| P4 | Modernization | `02-engineering/migrations/*` guides |
| P5 | Design Audit & Handover | `01-core/prompts/05-design-audit.md`, `06-design-handover.md`, `07-design-redesign.md`, `08-implementation-handover.md` |
| P6 | Redesign Implementation | `01-core/prompts/09-ui-implementation.md`, `10-icon-redesign.md`, `11-screenshot-generation.md` |
| P7 | ASO & Store Assets | `01-core/prompts/12-aso-audit.md`, `13-keyword-research.md`, `14-store-assets.md` |
| P8 | Verification & Release | `01-core/prompts/15-final-verification.md`, `16-release-preparation.md` |

## Phase flow

Happy path:

```text
P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8 → RELEASE → post-release operations (ongoing loop)
```

Gates and loop-backs:

| Phase | Gate (summary — full criteria in the phase section) | On pass → | On fail → loop back to |
|---|---|---|---|
| P1 Discovery | DISCOVERY.md complete; build attempted or failure documented | P2 | — (P1 is the entry point; later phases loop back here on discovery gaps) |
| P2 Project Memory | State file + changelog instantiated, zero `{{...}}` tokens, self-verification passed | P3 | P1 — missing fact: amend DISCOVERY.md, never guess |
| P3 Engineering Audit & Roadmap | Findings evidenced; risk register created; roadmap operator-accepted | P4 | P1 — discovery gap found during audit |
| P4 Modernization | Build verification + `engineering-modernization` checklist pass | P5 | P3 — blocked migration step: re-plan remaining roadmap only |
| P5 Design Audit & Handover | All handover documents validate against schemas | P6 | P1 — discovery gap; engineering debt goes to Backlog, not fixed in P5 |
| P6 Redesign Implementation | Build verification + `design-modernization` + `accessibility` checklists pass | P7 | P5 — defective/ambiguous handover: revise as `_v<N+1>` |
| P7 ASO & Store Assets | `aso-launch` checklist; metrics sourced; assets meet store specs | P8 | within P7 — revise copy/assets; feature gaps go to Backlog |
| P8 Verification & Release | `release-readiness` checklist; release build device-verified | RELEASE, then post-release operations | P4 (code defect) / P6 (UI defect) / P7 (asset or listing defect), then directly back to P8 |

Loop-backs are scoped repairs, never restarts: a failed gate returns to the phase that owns the
defective artifact, fixes it, and fast-forwards back through already-passed gates by re-running
only their verification steps.

---

## P1 — Discovery

**Purpose.** Build a complete factual picture of an unknown (or stale) codebase: what the app
is, how it is built, what it depends on, where the risks are. P1 produces facts, not judgments.

**Entry criteria.** Target repo identified and readable; operator has named the app and the
engagement goal. Nothing else — P1 is the universal entry point.

**Activities.** Repository census (modules, entry points, manifest); build attempt; dependency,
SDK, and service inventories; manifest audit; identification of the app's category playbook in
`06-playbooks/` if one matches.

**Prompts & factory docs used.** `01-core/prompts/01-discovery.md`;
`02-engineering/01-discovery.md`, `02-architecture-mapping.md`, `03-dependency-analysis.md`,
`04-sdk-inventory.md`, `05-service-inventory.md`, `06-manifest-audit.md`;
`08-knowledge/android/version-matrix.md` for dating the stack.

**Artifacts produced.** `docs/factory/audits/DISCOVERY.md` — P1's only artifact.
PROJECT_STATE.md is created in P2, never in P1.

**Exit criteria / quality gate.** DISCOVERY.md exists and accounts for every Gradle module,
every manifest entry point, and every third-party SDK; the app builds via the one optional
`assembleDebug` attempt, or the build failure is documented with its cause — never fixed in P1
(build policy: `01-core/CONVENTIONS.md`); every claim cites a file path.

**Typical duration.** 1–2 Claude sessions (1 for a single-module app; 2 with per-module
subagent fan-out for multi-module apps).

**Loop-back rules.** None into P1 from earlier (it is first). P3 or P5 may loop back to P1 when
an audit reveals discovery gaps (an unexamined module, an undocumented flavor); the loop-back
amends DISCOVERY.md, then control returns to the phase that found the gap.

---

## P2 — Project Memory

**Purpose.** Instantiate the project's persistent memory so every future session can bootstrap
without re-discovery. P2 turns the discovery facts into the living state file and changelog.

**Entry criteria.** P1 Done: `docs/factory/audits/DISCOVERY.md` passed its gate.

**Activities.** Instantiate `01-core/PROJECT_STATE_TEMPLATE.md` →
`docs/factory/PROJECT_STATE.md`, filling every token from DISCOVERY.md and operator input;
instantiate `01-core/CHANGELOG_TEMPLATE.md` → `docs/factory/CHANGELOG.md`; backfill the Phase
Tracker — P1 Done, and P2 **In Progress until the prompt's final self-verification passes**,
only then Done; create `docs/factory/` subdirectories (`audits/`, `handovers/`, `reports/`,
`assets/`).

**Prompts & factory docs used.** `01-core/prompts/02-project-memory.md`;
`01-core/PROJECT_STATE_TEMPLATE.md`; `01-core/CHANGELOG_TEMPLATE.md`.

**Artifacts produced.** `docs/factory/PROJECT_STATE.md`; `docs/factory/CHANGELOG.md`.

**Exit criteria / quality gate.** No `{{...}}` token remains in either instance; every fact in
PROJECT_STATE.md is traceable to DISCOVERY.md or a recorded operator answer; Phase Tracker and
Current Status are accurate; first CHANGELOG entry written; prompt 02's final self-verification
passed (only then does P2 flip from In Progress to Done).

**Typical duration.** 1 Claude session.

**Loop-back rules.** If tokens cannot be filled because DISCOVERY.md lacks the fact, loop back
to P1 to amend it — do not guess values into the state file.

---

## P3 — Engineering Audit & Roadmap

**Purpose.** Judge the codebase against the modern baseline and produce a sequenced, estimated
modernization roadmap the operator can approve.

**Entry criteria.** P2 Done: PROJECT_STATE.md and CHANGELOG.md exist and are current.

**Activities.** Gap analysis of SDK/Kotlin/AGP/Gradle/libraries against
`08-knowledge/android/modern-stack.md` and `version-matrix.md`; architecture and code-health
findings with Severity/Effort; risk-register creation — prompt 03 instantiates
`docs/factory/reports/RISK_REGISTER.md` from `09-templates/risk-register.md`, and prompt 04
adds workstream risks; roadmap generation sequencing migrations into safe order; operator
review of the roadmap. The risk register is reviewed at every subsequent phase gate.

**Prompts & factory docs used.** `01-core/prompts/03-modernization-audit.md`,
`01-core/prompts/04-roadmap-generation.md`; `02-engineering/03-dependency-analysis.md`,
`04-sdk-inventory.md`; `08-knowledge/android/common-pitfalls.md`;
`09-templates/engineering-audit-report.md`, `09-templates/risk-register.md`.

**Artifacts produced.** `docs/factory/audits/ENGINEERING_AUDIT.md`;
`docs/factory/audits/deps-<config>.txt`; `docs/factory/reports/MODERNIZATION_ROADMAP.md`;
`docs/factory/reports/RISK_REGISTER.md` (created by prompt 03); updated Backlog in
PROJECT_STATE.md.

**Exit criteria / quality gate.** Every finding has severity, effort, and cited evidence; the
roadmap orders work with dependencies explicit (e.g., AGP before Kotlin where required) and is
explicitly accepted by the operator (acceptance recorded in the Decision Log).

**Typical duration.** 1–2 Claude sessions.

**Loop-back rules.** Discovery gaps → P1 (amend DISCOVERY.md, return). If P4 later proves a
roadmap step infeasible, P4 loops back here for re-planning of the remaining sequence only.

---

## P4 — Modernization

**Purpose.** Execute the approved roadmap: bring build system, language, SDK targets, and key
libraries to the modern baseline while keeping the app shippable at every step.

**Entry criteria.** P3 Done: operator-accepted roadmap exists. Build is green (or its failure is
the first roadmap item).

**Activities.** Execute migrations strictly per the relevant guides — Gradle, AGP, Kotlin,
Android SDK — one step at a time, build-verifying between steps; dependency upgrades; kapt→KSP;
version-catalog adoption; regression checks on core flows; migration report.

**Prompts & factory docs used.** `02-engineering/migrations/gradle-migration.md`,
`agp-migration.md`, `kotlin-migration.md`, `android-sdk-migration.md`;
`02-engineering/07-build-verification.md`; `07-checklists/engineering-modernization.md`;
`08-knowledge/android/common-pitfalls.md`.

**Artifacts produced.** Migrated code (committed per step); `docs/factory/reports/
MODERNIZATION_REPORT.md`; updated Tech Stack Snapshot in PROJECT_STATE.md.

**Exit criteria / quality gate.** `02-engineering/07-build-verification.md` passes in full;
`07-checklists/engineering-modernization.md` complete; app installs and core flows work on
device/emulator; Tech Stack Snapshot updated to actual versions.

**Typical duration.** 2–6 Claude sessions, scaling with stack age and module count. Always
single-threaded (CLAUDE_MASTER Section 4).

**Loop-back rules.** A blocked migration step → P3 to re-plan remaining roadmap (completed steps
stand). Never loop forward: P5 must not start until the P4 gate passes, because design work
against a broken build produces unverifiable handovers.

---

## P5 — Design Audit & Handover

**Purpose.** Audit the app's UI/UX against Material 3 and modern mobile UX standards, design the
target state, and encode it in validated handover documents an implementing engineer can execute
without design judgment calls.

**Entry criteria.** P4 Done (full engagement) or modular-entry minimum met (see "Running phases
out of order"). App builds and runs so screens can be captured.

**Activities.** Screen inventory and baseline screenshot capture; design audit per screen
(subagent fan-out per CLAUDE_MASTER Section 4); design-system extraction; DESIGN_HANDOVER
authoring; redesign direction via prompt 07 (pasted into Claude Design, producing the
REDESIGN_PROPOSAL); IMPLEMENTATION_HANDOVER authoring from the proposal; validation of all
three handover documents against their schemas.

**Prompts & factory docs used.** `01-core/prompts/05-design-audit.md`, `06-design-handover.md`,
`07-design-redesign.md`, `08-implementation-handover.md`; `03-design/01-design-audit.md` through
`09-accessibility-review.md`; `05-handover/DESIGN_HANDOVER_SCHEMA.md`,
`REDESIGN_PROPOSAL_SCHEMA.md`, `IMPLEMENTATION_HANDOVER_SCHEMA.md`;
`07-checklists/handover-validation.md`; `08-knowledge/design/material3-essentials.md`,
`mobile-ux-principles.md`; `09-templates/design-audit-report.md`,
`09-templates/screen-inventory.md`.

**Artifacts produced.** `docs/factory/audits/SCREEN_INVENTORY.md` (its own file, never embedded
in the audit); `docs/factory/audits/DESIGN_AUDIT.md`; baseline screenshots under
`docs/factory/assets/screenshots/baseline/`; `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md`;
`docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md`;
`docs/factory/handovers/IMPLEMENTATION_HANDOVER_v<N>.md`.

**Exit criteria / quality gate.** All three handover documents pass
`07-checklists/handover-validation.md` against their `05-handover/` schemas; every audited
screen appears in the handover (redesigned or explicitly out of scope); accessibility review
included.

**Typical duration.** 2–4 Claude sessions (audit 1–2, redesign + handovers 1–2).

**Loop-back rules.** Engineering debt found during audit → recorded in PROJECT_STATE.md Backlog
for P3/P4 triage, not fixed in P5. P6 rejects a defective handover back to P5; the fix bumps the
handover version (`_v<N+1>`), and the superseded version is retained.

---

## P6 — Redesign Implementation

**Purpose.** Implement the validated handovers in code: screens, theme, components, icon,
splash — producing the modernized UI and fresh screenshots.

**Entry criteria.** P5 Done: handovers validated. Build green.

**Activities.** UI implementation per IMPLEMENTATION_HANDOVER, screen by screen; icon redesign
and asset generation; splash update; screenshot regeneration; per-screen verification against
the handover's acceptance specs; deviation log for anything that could not be built as specified.

**Prompts & factory docs used.** `01-core/prompts/09-ui-implementation.md`,
`10-icon-redesign.md`, `11-screenshot-generation.md`; `03-design/05-material3-modernization.md`,
`06-icon-redesign.md`, `07-splash-redesign.md`, `08-component-library.md`;
`07-checklists/design-modernization.md`, `07-checklists/accessibility.md`;
`02-engineering/07-build-verification.md`.

**Artifacts produced.** Implemented UI code (committed) with per-ticket status table; redesigned
screenshots under `docs/factory/assets/screenshots/redesigned/`; icon assets with store exports
under `docs/factory/assets/store/<store>/` and the brief in `docs/factory/reports/ICON_SPEC.md`;
store uploadables under `docs/factory/assets/store/<store>/<locale>/` with
`docs/factory/reports/SCREENSHOT_MANIFEST.md`.

**Exit criteria / quality gate.** Build verification passes; every handover item is Done or
Deferred-with-reason; `07-checklists/design-modernization.md` and the accessibility checklist
complete; before/after screenshots captured.

**Typical duration.** 2–6 Claude sessions, scaling with screen count.

**Loop-back rules.** Handover ambiguity or infeasibility → back to P5 for a handover revision
(`_v<N+1>`), **not** to scratch and **not** improvised in P6. P8 may loop back to P6 for UI
defects found in final verification.

---

## P7 — ASO & Store Assets

**Purpose.** Maximize store visibility and conversion: audit current listings, research
keywords, and produce per-store metadata and visual assets ready for upload.

**Entry criteria.** P1 + P2 minimum (modular entry). For full engagements, P6 Done so
screenshots reflect the new UI. App Identity section of PROJECT_STATE.md filled in.

**Activities.** Per-store ASO audit (fan out per store); keyword research and keyword map;
competitor analysis; store-listing copy per store; screenshot and icon optimization for store
context; localization plan where relevant; ASO handover.

**Prompts & factory docs used.** `01-core/prompts/12-aso-audit.md`, `13-keyword-research.md`,
`14-store-assets.md`; `04-aso/stores/google-play.md`, `huawei-appgallery.md`, `rustore.md`;
`04-aso/workflows/keyword-research.md`, `competitor-analysis.md`, `screenshot-optimization.md`,
`icon-optimization.md`, `localization.md`; `08-knowledge/aso/ranking-factors.md`,
`conversion-optimization.md`; `08-knowledge/stores/policy-landmines.md`;
`09-templates/aso-audit-report.md`, `keyword-map.md`, `store-listing.md`;
`05-handover/ASO_HANDOVER_SCHEMA.md`.

**Artifacts produced.** `docs/factory/audits/ASO_AUDIT.md`;
`docs/factory/reports/KEYWORD_MAP.md`;
`docs/factory/reports/STORE_LISTING_<store>_<locale>.md` per store/locale; store-ready assets
under `docs/factory/assets/store/<store>/<locale>/`;
`docs/factory/handovers/ASO_HANDOVER_v<N>.md`.

**Exit criteria / quality gate.** `07-checklists/aso-launch.md` complete; every metric sourced
or labeled `(estimate)`; assets meet each store's published specs; ASO handover validates
against its schema; no policy landmine left unaddressed.

**Typical duration.** 2–3 Claude sessions (1 per store plus synthesis, or audit/keywords/assets
split).

**Loop-back rules.** Listing claims that the app cannot substantiate → feature gap recorded in
Backlog (P3/P4 territory), copy revised to honest claims within P7. P8 loops back to P7 for
asset or listing defects.

---

## P8 — Verification & Release

**Purpose.** Prove the app and its listing are ready, then prepare the release: final
verification across code, UI, and store assets, release build, and release handover.

**Entry criteria.** All in-scope phases Done. For a full engagement: P4, P6, P7 gates passed.
For modular engagements: the in-scope subset passed plus P1+P2.

**Activities.** Final verification sweep (build, install, core flows, regressions, asset
completeness); release-readiness checklist; version bump per policy; release build and signing
coordination (signing requires operator — escalation rule); release notes; release report;
RELEASE handover; post-release monitoring plan.

**Prompts & factory docs used.** `01-core/prompts/15-final-verification.md`,
`16-release-preparation.md`; `07-checklists/release-readiness.md`;
`10-releases/release-workflow.md`, `app-versioning-policy.md`;
`05-handover/RELEASE_HANDOVER_SCHEMA.md`; `09-templates/verification-report.md`,
`09-templates/release-report.md`; `02-engineering/07-build-verification.md`.

**Artifacts produced.** Verified release build (AAB/APK — path recorded, binary not committed);
`docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (re-runs append `_r2`, `_r3`);
`docs/factory/reports/RELEASE_REPORT_v<versionName>.md`;
`docs/factory/handovers/RELEASE_HANDOVER_v<N>.md`; updated Store Presence and Release History
in PROJECT_STATE.md after publication.

**Exit criteria / quality gate.** `07-checklists/release-readiness.md` fully passed; release
build verified on device/emulator; versioning conforms to
`10-releases/app-versioning-policy.md`; RELEASE handover validates; operator has everything
needed to press "publish".

**Typical duration.** 1–2 Claude sessions.

**Loop-back rules.** Verification failure routes by defect owner: code defect → P4; UI defect →
P6; asset/listing defect → P7. After the fix, return directly to P8 and re-run only the failed
verification plus a regression spot-check — do not re-execute the intermediate phases.

---

## After P8 — post-release operations

Release is not the end of the lifecycle; it opens an ongoing operations loop. After
publication, work continues per `04-aso/workflows/post-launch-monitoring.md` (staged-rollout
monitoring, vitals watch, review response, listing experiments) gated by
`07-checklists/aso-post-release.md`. Experiments are logged in
`docs/factory/reports/EXPERIMENT_LOG.md` and KPI snapshots land in PROJECT_STATE.md. Findings
from this loop feed the Backlog and can open a new engagement at any phase via modular entry.

---

## Running phases out of order (modular entry)

The factory supports partial engagements, but never blind ones. Rules:

1. **P1 and P2 are mandatory for every engagement**, regardless of scope. No phase may run
   against an app with no DISCOVERY.md and no PROJECT_STATE.md. They may be run in a compressed
   form (single session covering both) for narrow engagements, but their gates still apply.
2. **Common modular entries:**

   | Engagement | Entry phase | Minimum prerequisite phases |
   |---|---|---|
   | ASO-only | P7 | P1 + P2 |
   | Design-refresh-only | P5 → P6 | P1 + P2 (P4 strongly recommended; record the skip as a risk) |
   | Modernization-only | P3 → P4 | P1 + P2 |
   | Release-assist-only | P8 | P1 + P2, plus evidence the build is green |

3. **Skipped phases are recorded**, not erased: mark them Deferred in the Phase Tracker with the
   reason, and record the scope decision in the Decision Log. A later engagement can convert
   Deferred to In Progress.
4. **Gates are not waivable by modularity.** An ASO-only engagement entering at P7 still cannot
   exit P7 without the P7 gate. Entry shortcuts exist; exit shortcuts do not.
5. **Out-of-order hazards**: P6 without P4 risks building new UI on deprecated toolchains
   (record as Known Risk); P7 before P6 produces store assets showing the old UI (acceptable
   only if no redesign is in scope).

## Phase status tracking

Phase state lives in exactly one place: the **Phase Tracker** table of
`docs/factory/PROJECT_STATE.md` (instantiated from `01-core/PROJECT_STATE_TEMPLATE.md`), one row
per phase P1–P8.

- **Status values** use the canonical Status scale defined in `01-core/CONVENTIONS.md`
  (Not Started … Deferred). No other values are valid.
- **Done** may be set only when this document's exit criteria for that phase are met — the same
  rule as CLAUDE_MASTER Section 3, rule 2. Setting Done is always accompanied by the artifact
  link in the tracker row and a CHANGELOG entry naming the gate.
- **Blocked** requires a named blocker in Current Status; **Deferred** requires a Decision Log
  entry.
- **Loop-backs** reopen a phase: set the reopened phase to In Progress, leave its previous
  artifacts in place (handover revisions bump `_v<N>`), and note the loop-back in both the
  Session Log and the CHANGELOG entry. Phases that were Done and were not the loop-back target
  stay Done.
- The **Current Status** block (current phase / status / next action / blockers) must always
  agree with the tracker. The Session Bootstrap Procedure (CLAUDE_MASTER Section 2) reads both
  and treats the state file as truth; keeping them consistent is part of every session's end
  procedure.
