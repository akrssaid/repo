# WORKED_EXAMPLE — NotaFast, P1 → P8 in Miniature

This file traces one fictional app through the full lifecycle so a new agent or operator can see
what conformant factory output actually looks like. Every excerpt below is written exactly as the
real artifact would be — IDs per `01-core/CONVENTIONS.md` Section 1, paths per Section 5, scales
per Section 2 — at perhaps 5% of real length. Use it two ways: read it once to understand the
flow, and copy its shapes when unsure whether your own artifact conforms.

**The target.** NotaFast is a note-taking/document app, package `com.example.notafast`, one
operator (the publisher), published on Google Play. Legacy stack: minSdk 21 / targetSdk 31,
100% Java + XML layouts, AGP 7.2.2, Gradle 7.3.3, versionName 1.8.3 (versionCode 47). The
engagement is a full lifecycle: modernize, redesign, re-list, release 2.0.0. The fictional
engagement runs 2026-03-02 → 2026-04-24 over 14 sessions.

---

## P1 — Discovery (Session 1, 2026-03-02)

The operator pastes the brand-new-project snippet from `QUICK_REFERENCE.md`, naming the target
repo. The agent loads `01-core/CLAUDE_MASTER.md`, finds no `docs/factory/`, and runs
`01-core/prompts/01-discovery.md`. NotaFast has 2 modules (≤ 3), so discovery is
single-threaded. Per the P1 build policy (`CONVENTIONS.md` Section 9) the agent runs read-only
Gradle plus one `assembleDebug` attempt — which fails on JDK 17 vs AGP 7.2.2; the failure is
documented, not fixed. The only artifact written is `docs/factory/audits/DISCOVERY.md`:

```markdown
## Summary
NotaFast (com.example.notafast) is a Java/XML note-taking and document app: 2 Gradle
modules (:app, :pdfcore), 34 layout XMLs, 0 composables, 6 activities. Stack dates to
2022 (AGP 7.2.2, targetSdk 31). One assembleDebug attempt FAILED on host JDK 17
(AGP 7.2.2 requires JDK 11) — documented per P1 build policy, not fixed.
Discovered 2026-03-02 by factory P1 (prompt 01).

## Build System
| Component | Version | Evidence |
|---|---|---|
| Gradle | 7.3.3 | gradle/wrapper/gradle-wrapper.properties:3 |
| AGP | 7.2.2 | build.gradle:9 |
| Kotlin | none — Java-only | Inference: no kotlin plugin in any build file |
| minSdk / targetSdk | 21 / 31 | app/build.gradle:14-15 |
```

**Gate.** Every module, manifest entry point, and SDK accounted for; build failure documented
with cause; claims cited `path:line`. The operator confirms; P1 is Done.

---

## P2 — Project Memory (Session 2, 2026-03-03)

The agent runs `01-core/prompts/02-project-memory.md`: instantiates
`01-core/PROJECT_STATE_TEMPLATE.md` → `docs/factory/PROJECT_STATE.md` and
`CHANGELOG_TEMPLATE.md` → `docs/factory/CHANGELOG.md` per the instantiation doctrine
(`CONVENTIONS.md` Section 11) — copy body, fill every token, delete guidance, grep `{{` = empty.
Note the P2 status rule: P2 shows In Progress until the prompt's final self-verification passes.
Mid-session, the Phase Tracker reads:

```markdown
## Phase Tracker
| Phase | Name | Status | Date | Artifact |
|---|---|---|---|---|
| P1 | Discovery | Done | 2026-03-02 | docs/factory/audits/DISCOVERY.md |
| P2 | Project Memory | In Progress | 2026-03-03 | docs/factory/PROJECT_STATE.md |
| P3 | Engineering Audit & Roadmap | Not Started | — | — |
| P4 | Modernization | Not Started | — | — |
| P5 | Design Audit & Handover | Not Started | — | — |
| P6 | Redesign Implementation | Not Started | — | — |
| P7 | ASO & Store Assets | Not Started | — | — |
| P8 | Verification & Release | Not Started | — | — |

## Tech Stack Snapshot
| Component | Current | Evidence |
|---|---|---|
| AGP | 7.2.2 | build.gradle:9 |
| CI | none — no workflow files found | DISCOVERY.md §Build Variants, Signing & CI |
```

Self-verification passes (zero `{{` tokens, every fact traceable to DISCOVERY.md or a recorded
operator answer), the P2 row flips to Done, and the first CHANGELOG entry is appended.

---

## P3 — Engineering Audit & Roadmap (Sessions 3–4, 2026-03-05 / 03-06)

Prompt 03 judges the stack against `08-knowledge/android/modern-stack.md`, writes
`docs/factory/audits/ENGINEERING_AUDIT.md` from the `engineering-audit-report` template, dumps
`audits/deps-debugRuntimeClasspath.txt`, and **creates** `reports/RISK_REGISTER.md` from the
risk-register template (P3 owns risk-register creation — `CONVENTIONS.md` Section 12). Two of
the eleven findings:

```markdown
| ID | Finding | Severity | Effort | Evidence |
|---|---|---|---|---|
| ENG-003 | targetSdk 31 is below the Play floor (API 35 now; API 36 at the
  Aug 2026 deadline) — app cannot ship updates without bump | Critical | L |
  app/build.gradle:15 |
| ENG-007 | Six AsyncTask call sites (deprecated since API 30, not removed);
  document-export path blocks UI thread on large PDFs | High | M |
  app/src/main/java/com/example/notafast/export/ExportTask.java:41 |
```

Prompt 04 then sequences the findings into `reports/MODERNIZATION_ROADMAP.md` as lettered
workstreams (and adds workstream risks to the risk register):

```markdown
### WS-A — Build toolchain to modern baseline
Scope: Gradle 7.3.3→8.11.1, AGP 7.2.2→8.9.1 (compileSdk 36 requires AGP ≥ 8.9.1),
JDK 17 toolchain. Sequencing: FIRST — every other workstream builds on this.
Findings: ENG-001, ENG-002, ENG-003. Effort: L. Risk: RISK-002 (plugin breakage, M/H).
Exit: Tier 1–2 verification per 02-engineering/07-build-verification.md.
```

**Gate.** Every finding has severity/effort/evidence; the operator accepts the roadmap and the
acceptance is recorded in the Decision Log. P3 Done.

---

## P4 — Modernization (Sessions 5–7, 2026-03-09 → 03-16)

No dedicated prompt: the agent executes `02-engineering/migrations/gradle-migration.md`,
`agp-migration.md`, then `android-sdk-migration.md`, single-threaded, one step at a time on
branch `factory/p4-toolchain`, build-verifying between steps. Commits follow the canonical
format, e.g.:

```
build(gradle): P4 WS-A upgrade AGP 7.2.2 -> 8.9.1, Gradle wrapper 8.11.1
fix(app): P4 WS-B replace ExportTask AsyncTask with ExecutorService
```

End state: AGP 8.9.1, Gradle 8.11.1, compileSdk 36, targetSdk 35, minSdk 23 (operator decision,
logged), still Java+XML (Kotlin adoption deferred to Backlog — Deferred needs a Decision Log
entry, and gets one). `reports/MODERNIZATION_REPORT.md` written; Tech Stack Snapshot updated;
`07-checklists/engineering-modernization.md` passes; app installs and core flows verified on
emulator. P4 Done.

---

## P5 — Design Audit & Handover (Sessions 8–10, 2026-03-18 → 03-25)

**Prompt 05** inventories screens into `audits/SCREEN_INVENTORY.md` (a separate file, never
embedded in the audit) and captures baselines into `assets/screenshots/baseline/` — demo clock
10:00, dark captured for every screen, file names `SCR-<id>-<state>-<theme>.png`:

```markdown
| ID | Screen | Entry point | States present | Baseline captures |
|---|---|---|---|---|
| SCR-001 | Note List (home) | MainActivity | default, empty, loading, dark
  (error, offline: N/A — local-only screen) | SCR-001-default-light.png,
  SCR-001-default-dark.png, SCR-001-empty-light.png, SCR-001-loading-light.png |
| SCR-002 | Editor | EditorActivity (intent from SCR-001) | default, error, dark |
  SCR-002-default-light.png, SCR-002-default-dark.png, SCR-002-error-light.png |
```

`audits/DESIGN_AUDIT.md` scores each screen against the H1–H9 rubric owned by
`03-design/01-design-audit.md`. One per-screen finding:

```markdown
| ID | Screen | Finding | Heuristic | Severity | Effort |
|---|---|---|---|---|---|
| DES-004 | SCR-001 | Empty state is a bare TextView ("No notes") with no
  illustration or action; first-run users see a dead end | H6 | High | S |
```

**Prompt 06** compiles `handovers/DESIGN_HANDOVER_v1.md` against
`05-handover/DESIGN_HANDOVER_SCHEMA.md`. Its Asset Manifest (the canonical name — never
"Assets Bundle") and one per-screen brief:

```markdown
## Asset Manifest
| ID | Asset | Format | Path | Status |
|---|---|---|---|---|
| ASSET-001 | Launcher icon source | SVG | docs/factory/assets/store/google-play/icon-src.svg | Delivered |
| ASSET-002 | Brand color tokens | Markdown table (hex) | §Design Tokens below | Delivered |
| ASSET-003 | Empty-state illustration, note list | SVG | docs/factory/assets/screenshots/baseline/SCR-001-empty-ref.svg | Requested — operator to approve style |

## Per-Screen Brief — SCR-001 Note List
Current: dense ListView, legacy holo accent #FF4081, bare empty state (DES-004).
Target: M3 list with NoteCard component, tonal surface containers, FAB → editor.
States to design: default, empty (with ASSET-003 + action button), loading, + dark.
Frozen strings: app name "NotaFast", folder names (user data). Redesignable: empty-state
copy, FAB content description (see §Localization & Content Constraints).
```

**Prompt 07 is pasted INTO Claude Design** together with the handover; it returns
`handovers/REDESIGN_PROPOSAL_v1.md` (validated against `REDESIGN_PROPOSAL_SCHEMA.md`). Its
SCR-001 section carries a Motion line using the vocabulary owned by
`08-knowledge/design/material3-essentials.md`:

```markdown
Motion: SCR-001 note card → SCR-002 editor opens via container transform, 300ms
(medium), emphasized easing; reduced-motion fallback: cross-fade 150ms.
```

**Prompt 08** converts the proposal into `handovers/IMPLEMENTATION_HANDOVER_v1.md`. Every
proposal Motion line maps to a ticket or is explicitly rejected; copy changes become COPY
tickets (no blanket string freeze — but un-ticketed string changes stay banned):

```markdown
### IMPL-006 — Note card → editor container transform
Screen: SCR-001 → SCR-002. Source: REDESIGN_PROPOSAL_v1 §SCR-001 Motion.
Spec: MaterialContainerTransform on card tap; return transition mirrors.
Motion: container transform, 300ms, emphasized easing; reduced-motion: cross-fade 150ms.
A11y criteria: transition respects ANIMATOR_DURATION_SCALE=0; focus lands on editor title.
Effort: M. Status: Not Started.

### COPY-002 — Note list empty-state copy
Screen: SCR-001. Source: REDESIGN_PROPOSAL_v1 §SCR-001 empty state.
Change: "No notes" → "No notes yet — tap + to capture your first thought."
Constraint: string is declared redesignable in DESIGN_HANDOVER_v1 §Localization &
Content Constraints; update all 3 locales. Effort: S. Status: Not Started.
```

**Gate.** All three documents pass `07-checklists/handover-validation.md` against their
schemas; the operator marks each **Accepted** — from that moment they are immutable except the
Rejection Log and Validation Appendix (`CONVENTIONS.md` Section 2). P5 Done.

---

## P6 — Redesign Implementation (Sessions 11–12, 2026-03-30 → 04-06)

On branch `factory/p6-redesign`, prompt 09 implements tickets in handover order — including the
splash ticket sourced from `03-design/07-splash-redesign.md` and both excerpted tickets above —
commits like `feat(ui): P6 IMPL-006 note card container transform`, and recaptures every screen
into `assets/screenshots/redesigned/` (same naming, demo clock 10:00, dark always). One ticket
proves infeasible as specified (a Motion spec colliding with a legacy Fragment transaction), so
P5 is reopened: `IMPLEMENTATION_HANDOVER_v2.md` is issued with the revised spec, `_v1` is
retained, and P6 resumes against `_v2`. Prompt 10 executes the icon via
`03-design/06-icon-redesign.md` (brief in `reports/ICON_SPEC.md`); prompt 11 produces
`assets/store/google-play/en-US/` uploadables + `reports/SCREENSHOT_MANIFEST.md`. Session 12's
CHANGELOG entry, per `01-core/CHANGELOG_TEMPLATE.md`:

```markdown
## [2026-04-06] — Session 12 — Phase P6

**Role:** Principal Android Engineer (UI specialist)

### Added
- Container transform SCR-001 → SCR-002 per IMPL-006, with reduced-motion
  cross-fade fallback (commits 3f9a1c2, 8b04d7e)
- Redesigned captures for SCR-001..SCR-006 under
  docs/factory/assets/screenshots/redesigned/ (clock 10:00, dark included)

### Changed
- Empty-state copy per COPY-002 in values/, values-de/, values-ru/ (commit a91e44f)

### Handover
- Consumed docs/factory/handovers/IMPLEMENTATION_HANDOVER_v2.md — all 14 tickets
  Done, 1 Deferred with reason (IMPL-011, tablet pane — out of scope this release)

Build verification: PASSED
```

**Gate.** `design-modernization.md` + `accessibility.md` checklists pass; every ticket Done or
Deferred-with-reason; before/after screenshots captured. P6 Done.

---

## P7 — ASO & Store Assets (Session 13, 2026-04-13)

Prompt 12 writes `audits/ASO_AUDIT.md` (the only valid path — never `reports/`); prompt 13
produces `reports/KEYWORD_MAP.md` with clusters `KW-001` (note taking), `KW-002` (document
scanner), seeded per the volumes owned by `04-aso/workflows/keyword-research.md`. Prompt 14
instantiates the store-listing template — keyword density per
`08-knowledge/aso/ranking-factors.md`, every char count actually counted:

```markdown
## Title
| Field | Value | Chars |
|---|---|---|
| Title (en-US) | NotaFast: Fast Notes & Docs | 27/30 |
Rationale: leads with brand, covers KW-001 head term "notes"; "docs" bridges KW-002.
```

`handovers/ASO_HANDOVER_v1.md` carries the FINAL copy (variants live only in the STORE_LISTING
files and the EXP queue: `EXP-001`, an icon A/B for the experiment log), the mandatory Baseline
Metrics Snapshot, and the Category & tags rationale. It validates and is Accepted. P7 Done.

---

## P8 — Verification & Release (Session 14, 2026-04-20 → 04-24)

Prompt 15 runs the full sweep and writes `reports/VERIFICATION_REPORT_v2.0.0.md` — versioned by
the app's versionName, not a revision count (`CONVENTIONS.md` Section 7). One flaky check forces
a re-run, which appends: `VERIFICATION_REPORT_v2.0.0_r2.md`, verdict:

```markdown
Verdict: PASS with waivers
Waivers: W-1 — IMPL-011 (tablet pane) Deferred per IMPLEMENTATION_HANDOVER_v2;
tracked in Backlog. No other deviations.
```

Prompt 16 prepares versionName 2.0.0 / versionCode 48 per
`10-releases/app-versioning-policy.md`, tags `release/v2.0.0`, hands the unsigned bundle to the
operator (signing is always an escalation), and executes rollout stages 4–5 of
`10-releases/release-workflow.md` **by reference** — the stage/dwell/threshold numbers live
there, nowhere else. The rollout log in `reports/RELEASE_REPORT_v2.0.0.md` fills one row per
stage; the first:

```markdown
| Stage | Started | Dwell | Crash rate (baseline 0.30%) | ANR | Decision |
|---|---|---|---|---|---|
| 10% | 2026-04-21 09:00 | 26h | 0.31% (+0.01pp) | 0.08% | ADVANCE to 25% — operator approved |
```

`handovers/RELEASE_HANDOVER_v1.md` validates; Store Presence, Release History
(2.0.0 / 48 / 2026-04-24 / rollout completed 100%), and the Artifact Index are updated in
PROJECT_STATE.md; the Phase Tracker shows P1–P8 Done. The engagement closes with `EXP-001`
queued in `reports/EXPERIMENT_LOG.md` for post-release.

---

## Conformance map (what each excerpt demonstrates)

| Excerpt | Convention it demonstrates |
|---|---|
| DISCOVERY fragment | P1 sole-artifact rule + build policy (`CONVENTIONS.md` §4, §9) |
| Phase Tracker | Canonical Status scale + P2 In-Progress rule (§2, §6) |
| ENG-003 / ENG-007, WS-A | ID formats, Severity/Effort scales, evidence citation (§1, §2) |
| SCR rows | `States present` column, screenshot naming, dark-always, clock 10:00 (§8) |
| DES-004 | Per-screen H1–H9 scoring pointer (§12) |
| Asset Manifest + brief | ASSET IDs, "Asset Manifest" canonical name, frozen-strings declaration (§1, §13) |
| Motion line → IMPL-006 | Motion pipeline: proposal line maps to a ticket with `Motion:` + `A11y criteria:` fields |
| COPY-002 | COPY ticket discipline — no un-ticketed string changes (§13) |
| Title field | Actually-counted chars; density owned by ranking-factors.md (§12) |
| Rollout row | Stage numbers owned by release-workflow stage 4; operator advances stages (§12) |
| CHANGELOG entry | One entry per session, commits cited, verification line, handover linkage |
| `_v2` handover, `_r2` report | Versioning semantics: revisions vs versionName vs re-runs (§7) |
