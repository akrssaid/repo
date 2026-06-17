# Prompt 04 — Modernization Roadmap

**Lifecycle phase:** P3 — Engineering Audit & Roadmap

This prompt converts the findings register in `docs/factory/audits/ENGINEERING_AUDIT.md` into an
executable modernization plan: findings grouped into workstreams, workstreams sequenced in
mandatory dependency order (toolchain before SDK before libraries before architecture), each sized,
risk-rated, given a rollback note and a build-green verification gate, and rolled up into a
milestone plan. The full workstream detail lives in
`docs/factory/reports/MODERNIZATION_ROADMAP.md` (the rationale document); the living execution
copy is written into `docs/factory/PROJECT_STATE.md` *Backlog* in the template's own column
format. The roadmap is the sole work order for P4 — Modernization. Workstream risks are appended
to the risk register Prompt 03 created.

## Role

You are a Principal AI Systems Architect who designs multi-week modernization programs that agents
and humans execute incrementally without breaking the build. Your specialty is dependency-aware
sequencing: you know that upgrading libraries before the toolchain wastes every hour spent, and
that a workstream without a verification gate and a rollback note is a gamble, not a plan. You
plan only from audited evidence — if the audit didn't find it, it isn't on your roadmap.

## Objective

Produce a complete modernization roadmap in which every Critical and High finding from
ENGINEERING_AUDIT.md is mapped to exactly one workstream (or explicitly Deferred with written
rationale), workstreams are sequenced Gradle → AGP → Kotlin → SDK → libraries → architecture with
the reasoning stated — preceded by a characterization-test workstream when the audit found no test
safety net — each workstream carries size (S/M/L/XL), risk, rollback note, and a verification gate
per `02-engineering/07-build-verification.md`, the whole is grouped into dated milestones, the
PROJECT_STATE Backlog mirrors the plan in the template's columns, and the risk register gains the
workstream risks. Done means an agent can open the first workstream tomorrow and execute P4
without asking a single sequencing question.

## Preconditions & Required Inputs

| Requirement | Detail | If missing |
|---|---|---|
| Lifecycle phase | P3 (second half); Prompt 03 complete. | Run `01-core/prompts/03-modernization-audit.md` first — never plan from an unaudited codebase. |
| `docs/factory/audits/ENGINEERING_AUDIT.md` | The findings register (`ENG-xxx`) — sole input for scope. | Hard stop; this prompt invents nothing. |
| `docs/factory/PROJECT_STATE.md` | Current Tech Stack Snapshot, Backlog section to populate, operator constraints if recorded. | Hard stop; re-run P2. |
| `docs/factory/reports/RISK_REGISTER.md` | Created by Prompt 03; this prompt appends workstream risks to it. | Hard stop; re-run Prompt 03's step 10 — never create the register here. |
| Migration guides | `02-engineering/migrations/gradle-migration.md`, `agp-migration.md`, `kotlin-migration.md`, `android-sdk-migration.md` — referenced per workstream as the execution method. | Proceed; reference the paths anyway (P4 executors will need the factory checkout). |
| Testing method | `02-engineering/08-testing-strategy.md` — the method behind the characterization-test workstream (WS-0) when needed. | Proceed; reference the path anyway. |
| Verification method | `02-engineering/07-build-verification.md` + the exact build commands recorded by Prompt 03. | Define gates with standard commands (`./gradlew build lint test`) and mark `verify commands against project`. |
| Operator constraints | Release freezes, budget, "do not touch" areas — ask the operator once before planning if not recorded in PROJECT_STATE.md. | Plan unconstrained and flag the assumption in the roadmap header. |

## Agent Orchestration

**Stay single-threaded.** Sequencing is a global optimization over one findings list — parallel
planners produce conflicting orders and merge pain exceeds any speedup. Optional bounded
delegation: after the workstream list is final (end of step 3), you may fan out at most 3–4
subagents (well under the factory cap of 8 — `01-core/CONVENTIONS.md`), one per major workstream,
to draft the detail block (steps in the workstream, risk, rollback, gate) for that workstream,
each returning the fixed workstream block format shown in step 4. The orchestrator alone owns
ordering, milestone grouping, and both output documents. Skip even this for roadmaps with ≤ 6
workstreams.

## Procedure

1. **Ingest the audit.** Read ENGINEERING_AUDIT.md fully. Extract every finding into a working
   table: ID, title, severity, effort, theme. Read PROJECT_STATE.md for constraints (release
   plans, deadlines, operator notes). Verify finding IDs are unique and the severity counts match
   the audit's executive summary — if the audit is internally inconsistent, fix nothing; send it
   back (Failure Modes item 1).

2. **Group into workstreams.** Cluster findings by what is *fixed together*, not by what is
   *named alike*. Canonical workstream set (instantiate only those with findings; merge or split
   as the project demands; IDs `WS-A…` per the ID registry in `01-core/CONVENTIONS.md`):
   - **WS-0 Testing baseline** — **mandatory whenever the audit found no test safety net** (zero
     or near-zero tests covering the code the migrations will touch): characterization tests per
     `02-engineering/08-testing-strategy.md`, pinning current behavior of the riskiest surfaces
     (data layer, export/import, billing paths) **before** any migration workstream runs. A
     migration without a safety net converts every "Low risk" label below into fiction.
   - **WS-A Toolchain** — Gradle wrapper, AGP, JDK toolchain, version catalog adoption, repository
     hygiene (JCenter removal).
   - **WS-B Kotlin & processing** — Kotlin to 2.1+/K2, kapt→KSP, synthetics removal.
   - **WS-C SDK posture** — compileSdk/targetSdk to 36, edge-to-edge, new-SDK behavior changes,
     minSdk decision.
   - **WS-D Library currency** — AndroidX/Compose BOM/Play Services/Billing/vendor SDK upgrades;
     16 KB-ready native lib versions.
   - **WS-E Security & manifest** — exported components, cleartext, secrets remediation, backup
     rules, permission cleanup.
   - **WS-F Architecture & deprecated APIs** — AsyncTask→coroutines, SharedPreferences→DataStore,
     Activity Result API, back-handling, RxJava→Flow (if in scope).
   Every Critical/High finding lands in exactly one workstream. Medium/Low findings attach where
   convenient or go to a "Hygiene backlog" workstream. A finding that will *not* be addressed is
   marked **Deferred** with one sentence of rationale (e.g., "ENG-014 RxJava migration — Deferred:
   XL effort, no correctness risk, revisit after P6") — and a matching row in the risk register
   (its template requires every deferred finding to have one).

3. **Sequence by dependency order.** Order workstreams **WS-0 (when instantiated) → Gradle → AGP
   → Kotlin → SDK → libraries → architecture** (WS-E security items slot in wherever their
   prerequisites allow — pure manifest fixes can even go first). State this rationale explicitly
   in the roadmap, because P4 executors must not "helpfully" reorder:
   - **Tests before everything:** characterization tests must pin current behavior *before* the
     code starts moving, or they pin the bugs the migrations introduce
     (`02-engineering/08-testing-strategy.md`).
   - **Gradle before AGP:** every AGP version declares a minimum required Gradle version; AGP
     upgrades on old Gradle fail at configuration time (`02-engineering/migrations/gradle-migration.md`).
   - **AGP before Kotlin:** AGP releases are tested against specific Kotlin/KGP ranges, and KSP
     versions are pinned to Kotlin versions; toolchain mismatch errors mask real ones
     (`02-engineering/migrations/agp-migration.md`, `kotlin-migration.md`).
   - **Kotlin before SDK:** compileSdk 36 stubs and the libraries that accompany them assume a
     modern compiler; K2 + current KSP must be stable before absorbing SDK behavior changes
     (`02-engineering/migrations/android-sdk-migration.md`).
   - **SDK before libraries:** current AndroidX/Compose releases set `minCompileSdk` in their
     metadata — upgrading libraries first generates errors that the SDK bump would have prevented.
   - **Libraries before architecture:** architecture migrations (DataStore, coroutines, Activity
     Result API) should be written once, against final APIs — not written against old libraries
     and then churned again by the upgrades.
   Within a workstream, order steps the same way (e.g., inside WS-A: Gradle wrapper first, then
   AGP, then catalog adoption). Record any project-specific inversions with justification (rare;
   e.g., a Critical security fix ships before everything).

4. **Detail each workstream.** Produce one block per workstream — these blocks live **only** in
   the roadmap report (step 6), never in the Backlog:

   ```markdown
   ### WS-<X> — <Name>
   - **Findings covered:** ENG-003, ENG-007, ENG-011
   - **Method:** 02-engineering/migrations/<guide>.md (+ specific sections if known)
   - **Size:** S | M | L | XL  (= max of constituent finding efforts, +1 step if >4 findings)
   - **Risk:** Low | Medium | High — <one line: what could break, blast radius>
   - **Rollback:** <one concrete sentence — e.g., "Single revert commit; wrapper + catalog
     changes are atomic. Tag `pre-ws-a` before starting.">
   - **Verification gate (per 02-engineering/07-build-verification.md):**
     `./gradlew clean build lint test` green; app launches on API 26 and API 36 emulators;
     <workstream-specific check, e.g., "release minified build installs and opens">
   - **Depends on:** WS-<previous> passed its gate | none
   ```

   Rules: every workstream gets a git tag instruction in its rollback note; P4 executes each
   workstream on a `factory/p4-<slug>` branch with `<type>(<scope>): P4 <summary>` commits
   (git conventions: `01-core/CONVENTIONS.md`); no workstream's gate is weaker than "full build +
   lint green"; risk High requires an explicit smoke-test addition to the gate. Use Prompt 03's
   recorded build commands verbatim where available.

5. **Build the milestone plan.** Group sequenced workstreams into milestones sized for the
   operator's cadence (default: a milestone ≈ one focused week):
   - **M1 — Safety net & toolchain** (WS-0 if instantiated, WS-A, WS-B, quick WS-E items): the
     foundation milestone; after M1 the project builds on the modern toolchain with behavior
     pinned by tests.
   - **M2 — Platform** (WS-C, WS-D): SDK 36 posture and library currency; after M2 the app is
     Play-deadline-safe.
   - **M3 — Architecture & hygiene** (WS-F, remaining WS-E, hygiene backlog).
   Give each milestone a target date (absolute, e.g., "by 2026-06-26"), an exit criterion (all
   member gates passed), and the headline risk. If a Play deadline from the audit lands before a
   default date, pull the relevant workstream forward and say so. Milestone table shape:

   | Milestone | Workstreams | Target date | Exit criterion | Headline risk |
   |---|---|---|---|---|
   | M1 — Safety net & toolchain | WS-0, WS-A, WS-B, WS-E (quick items) | 2026-06-19 | All member gates green; tag `m1-done` | KSP processor parity with kapt output |
   | M2 — Platform | WS-C, WS-D | 2026-07-03 | Gates green; app runs on API 26 + 36 | SDK 36 behavior changes in background work |
   | M3 — Architecture & hygiene | WS-F, hygiene backlog | 2026-07-17 | Gates green; zero deprecated-API criticals remain | Regression surface of DataStore migration |

6. **Write `docs/factory/reports/MODERNIZATION_ROADMAP.md`.** Sections: `## Summary` (counts:
   findings in scope / deferred, workstreams, milestones, end state); `## Sequencing Rationale`
   (the step-3 ordering argument); `## Workstreams` (step-4 blocks in execution order);
   `## Milestone Plan` (step-5 table); `## Deferred Findings` (table: ID, title, severity,
   rationale, revisit trigger); `## Coverage Matrix` (every ENG-xxx → workstream or Deferred —
   one row per finding, no gaps).

7. **Write the Backlog into PROJECT_STATE.md.** Replace the Backlog section's rows with the
   living plan using **exactly the template's Backlog columns**
   (# / Item / Severity / Effort / Source phase / Status — `01-core/PROJECT_STATE_TEMPLATE.md`);
   do not add or rename columns. One row per workstream, in execution order; the WS reference
   leads the Item cell, and the full detail (gate, milestone, dependencies) stays in the roadmap
   report, linked once above the table:

   ```markdown
   Execution detail per workstream (gates, milestones, dependencies):
   `docs/factory/reports/MODERNIZATION_ROADMAP.md` § Workstreams.

   | # | Item | Severity | Effort | Source phase | Status |
   |---|---|---|---|---|---|
   | 1 | WS-0 — Testing baseline (characterization tests; M1) | High | M | P3 | Not Started |
   | 2 | WS-A — Toolchain: Gradle, AGP, catalog (ENG-001, ENG-004; M1) | Critical | M | P3 | Not Started |
   | 3 | WS-B — Kotlin & processing: K2, kapt→KSP (ENG-002, ENG-009; M1; after WS-A) | High | L | P3 | Not Started |
   ```

   Mapping rules: row order = execution order (# is the priority rank); Severity = highest
   severity among the workstream's covered findings; Effort = the workstream size; Source phase =
   `P3`; Status uses the canonical scale (`01-core/CONVENTIONS.md`). The Backlog is the copy P4
   updates as work proceeds (statuses only); the report is the rationale snapshot and is not
   edited during execution.

8. **Append workstream risks to the risk register.** For every workstream whose Risk line is
   Medium or High, and for every Deferred finding, add a row to
   `docs/factory/reports/RISK_REGISTER.md` (created by Prompt 03 — append with the next free
   `RISK-xxx` IDs, never renumber existing rows): event-form description, likelihood/impact,
   mitigation = the workstream's rollback + gate, trigger = the observable early warning (e.g.,
   "KSP-generated code diverges from kapt output in module X"). Update the risk matrix and the
   register's Last reviewed header. Sync the top risks into PROJECT_STATE.md *Known Risks*.

9. **Self-check and close.** Run the Coverage Matrix check: every Critical/High audit finding
   appears exactly once (workstream or Deferred-with-rationale). Check gate presence on every
   workstream. Check that WS-0 exists if the audit reported a missing test safety net. Check date
   sanity against 2026-06-10 and the next Play deadline. Diff the Backlog rows against the
   report's workstream list (same IDs, same order). Then apply Documentation Update Rules.

## Expected Outputs

| Artifact | Path (target repo) | Notes |
|---|---|---|
| Roadmap report | `docs/factory/reports/MODERNIZATION_ROADMAP.md` | Full rationale: sequencing argument, workstream blocks, milestones, coverage matrix. |
| Living backlog | `docs/factory/PROJECT_STATE.md` → *Backlog* section | One row per workstream in the template's exact columns; the copy P4 maintains. |
| Risk register update | `docs/factory/reports/RISK_REGISTER.md` | Workstream + deferred-finding risks appended (step 8); register was created by Prompt 03. |
| Changelog entry | `docs/factory/CHANGELOG.md` | Appended per Documentation Update Rules. |

No code, build files, or audit files are modified.

## Acceptance Criteria

- [ ] Coverage Matrix accounts for **every** finding ID in ENGINEERING_AUDIT.md; every Critical
      and High maps to a workstream or carries an explicit Deferred rationale — zero silent drops.
- [ ] If the audit found no test safety net, WS-0 (characterization tests per
      `02-engineering/08-testing-strategy.md`) exists and precedes every migration workstream;
      otherwise the roadmap states why a safety net already exists.
- [ ] Workstream order is (WS-0 →) Gradle → AGP → Kotlin → SDK → libraries → architecture, and
      the Sequencing Rationale section explains each arrow with the method/migration-guide
      reference.
- [ ] Every workstream block contains all seven fields (findings, method, size, risk, rollback
      with git-tag instruction, verification gate, depends-on) — no field reads "TBD".
- [ ] Every verification gate includes at minimum a full green build per
      `02-engineering/07-build-verification.md`, using the project's real build commands.
- [ ] Milestones carry absolute target dates and exit criteria; any audit-cited Play deadline is
      reflected in milestone ordering.
- [ ] PROJECT_STATE.md Backlog uses exactly the template's columns
      (# / Item / Severity / Effort / Source phase / Status), one row per workstream, same
      workstreams and order as the roadmap report, with the report linked above the table.
- [ ] RISK_REGISTER.md gained the workstream and deferred-finding risks with valid next-free
      `RISK-xxx` IDs, an updated matrix, and an updated Last reviewed header.
- [ ] Phase Tracker shows P3 = Done (audit + roadmap both complete) with date 2026-06-10.
- [ ] Both outputs contain zero `{{...}}` tokens and no findings that do not exist in the audit.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`:**
  - *Backlog*: replaced with the workstream rows (Procedure step 7). This is the only prompt that
    wholesale rewrites Backlog; from P4 onward only statuses and notes change.
  - *Phase Tracker*: P3 → `Done`, date 2026-06-10 (the audit half was left `In Progress` by
    Prompt 03). P4 row gains the note "execute Backlog top-down; gate per workstream".
  - *Known Risks*: sync the top workstream risks from the register (with `RISK-xxx` and WS IDs);
    keep audit risks already listed, now annotated with their workstream.
  - *Open Questions*: add any planning questions the operator must answer (constraints, freeze
    windows) that surfaced during sequencing.
  - *Artifact Index*: add the MODERNIZATION_ROADMAP.md row, date 2026-06-10; bump the
    RISK_REGISTER.md row's date.
- **`docs/factory/reports/RISK_REGISTER.md`:** append-only ID stream; matrix and review header
  updated (Procedure step 8). Never create it here — Prompt 03 owns creation.
- **`docs/factory/CHANGELOG.md`:** append (top) entry dated 2026-06-10, phase P3: "Modernization
  roadmap generated — N workstreams, M milestones, K findings deferred; risk register updated",
  with the report path. Append-only.

## Failure Modes & Recovery

1. **The audit is unusable as input** (missing IDs, uncited findings, internally inconsistent
   counts). Do not patch around it — a roadmap built on a broken audit inherits every defect.
   Return to `01-core/prompts/03-modernization-audit.md`, regenerate or repair the audit, then
   restart this prompt.
2. **Everything looks Critical, milestone M1 balloons.** Re-derive milestones from the *deadline*
   axis, not the severity axis: only findings with dated external consequences (Play deadlines,
   live security exposure) may enter M1. Severity says "must fix"; deadlines say "must fix first".
3. **Circular or ambiguous dependencies** (e.g., a library upgrade requires SDK 36, but the SDK
   bump's behavior changes require that library's fix). Split the workstream: take the minimum
   slice of the later workstream needed to unblock (a single library pinned to a bridge version),
   document the bridge step explicitly, and keep the global order intact.
4. **Operator constraints surface late** ("release freeze until July", "don't touch billing").
   Do not delete workstreams — re-tag affected ones Deferred or move them to a later milestone
   with the constraint quoted as rationale, record the decision in the Decision Log, and append a
   CHANGELOG entry recording the re-plan and its trigger.
5. **Sizing fantasy.** If a workstream aggregates 10+ findings and still claims size M, re-size:
   the workstream size is at least the max of its findings' efforts and grows with count.
   Optimistic sizing destroys milestone credibility and cascades through every date.
6. **Skipping WS-0 to "save a week".** A missing safety net is the single highest-leverage risk
   in the whole program: without characterization tests, every gate's "build green" proves
   compilation, not behavior. If the operator insists on skipping WS-0, record it as an operator
   decision in the Decision Log and add a High-likelihood risk row to the register — do not
   silently comply.
7. **Roadmap/Backlog drift** (you edited one and forgot the other). Before finishing, diff the
   workstream IDs and order between the report and the Backlog section; they must match exactly.
   If they ever diverge later in P4, the Backlog (living copy) wins and the discrepancy is logged
   in CHANGELOG.md.
