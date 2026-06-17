# RELEASE Handover Schema

This schema defines the contract for the release→operator handover: the document produced by
`01-core/prompts/16-release-preparation.md` (lifecycle phase P8) and consumed by two parties —
the human operator who uploads the artifact, drives the staged rollout, and makes hold/advance/
rollback calls, and the next-cycle agents (the next P1/P3 session) that read it as the
authoritative record of what shipped, how it behaved, and what was left for them. It is both a
runbook for the next two weeks and the permanent release record; everything in it must remain
true and verifiable after the producer's session is gone.

## File Naming & Versioning

- Instance path: `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` in the TARGET app repo.
- `<N>` starts at 1 *per app release cycle*; every revision bumps N. Never edit a handover after
  its status reaches Accepted — issue v(N+1) instead.
- Status lifecycle (recorded in the Handover Header): Draft → Delivered → Accepted | Rejected
  (canonical scale in `01-core/CONVENTIONS.md`).
- Exception to immutability: the Store Submission State and Sign-off Log sections are designed as
  living logs — appending dated rows to them during rollout is permitted and required. The
  `## Rejection Log` and `## Validation Appendix` sections are also appendable per
  `01-core/CONVENTIONS.md`. All other sections are frozen at acceptance.
- Must pass `07-checklists/handover-validation.md` and the
  `07-checklists/release-readiness.md` gate before the artifact is uploaded to any store.

## Required Sections

The document must contain exactly these 10 H2 sections, in this order. A missing or empty section
is an automatic validation failure. Three further sections are **whitelisted but not required**:
`## Rejection Log` and `## Validation Appendix` (appendable per `01-core/CONVENTIONS.md`); the
Store Submission State and Sign-off Log living logs grow in place. No other section may be added.

### 1. Handover Header

A key/value table with all of: App name, Package ID, Release version (versionName/versionCode),
Handover version (vN), Producer (`01-core/prompts/16-release-preparation.md`), Date produced
(absolute), Factory version, Status (Draft / Delivered / Accepted / Rejected), Supersedes
(vN-1 or "—"), Stores targeted.

### 2. Release Identity

The exact artifact being released — table with: versionName and versionCode (assigned per
`10-releases/app-versioning-policy.md`); the app release git tag in the canonical
`release/vX.Y.Z` format (per `01-core/CONVENTIONS.md` — e.g. `release/v3.2.0`; factory `factory/`
tags are a separate namespace) and the commit SHA it points to;
AAB path(s) (repo-relative or CI artifact URL) with SHA-256 checksum per file; signing config
identity (key alias / Play App Signing — never key material); build command used
(e.g. `./gradlew :app:bundleRelease`). Anyone must be able to confirm "this file is that release"
from this section alone.

### 3. Verification Summary

A link to the verification report produced by `01-core/prompts/15-final-verification.md`, named
`VERIFICATION_REPORT_v<versionName>.md` under `docs/factory/reports/` (re-runs append `_r2`, `_r3`;
e.g. `VERIFICATION_REPORT_v3.2.0.md`), the overall verdict (PASS / PASS with waivers / FAIL — the
word "CONDITIONAL" is abolished, per `01-core/CONVENTIONS.md`), and a table listing **every** gate
from
`07-checklists/release-readiness.md` with its result:

| Gate | Result (PASS / FAIL / WAIVED) | Evidence / waiver rationale + approver |
|---|---|---|

A release may only proceed on PASS, or PASS with waivers where every WAIVED row names a human
approver and rationale. Any unwaived FAIL makes the handover invalid (rule V3).

### 4. Change Summary

Two parts: **User-facing release notes per locale** — the exact "What's new" text per locale with
character count vs the store limit (Google Play: 500 per locale), final copy, paste-ready;
**Engineering changelog** — reference to the `docs/factory/CHANGELOG.md` entries this release
covers plus a bullet summary of notable internal changes (migrations, dependency bumps, anything
relevant to interpreting post-release vitals).

### 5. Rollout Plan

The staged-rollout stages, dwell times, and advance/HOLD thresholds are **owned by
`10-releases/release-workflow.md` stage 4** (the single source of truth per `01-core/CONVENTIONS.md`).
This handover does **not** restate those numbers — it executes stage 4 by reference. Likewise the
numeric Play-vitals bad-behavior thresholds live in
`08-knowledge/android/play-vitals-performance.md`; cite, never copy.

This section's job is to bind that workflow to **this** release: name the stage owner, confirm the
release uses the standard stage ladder (or document any approved deviation with approver), and map
each stage's advance gate to the metric source the operator will read.

| Stage (per release-workflow.md §4) | Advance gate source | Decision owner | Where the metric is read |
|---|---|---|---|
| As defined in `10-releases/release-workflow.md` stage 4 | `08-knowledge/android/play-vitals-performance.md` | Operator | Play Console → Vitals (+ Crashlytics fallback) |

Plus explicit **decision rules** for this release: the metric conditions under which the operator
holds (pause stage, investigate), advances, or rolls back — each rule stated as a reference to the
workflow's thresholds with a named owner. No stage may be advanced without the previous stage's
gate result recorded in the Sign-off Log.

### 6. Monitoring Plan

A table: what to watch (crash-free rate, ANR rate, vitals per Play Console / AppGallery Connect /
RuStore console, ratings velocity, key funnel metrics), where exactly (console + screen), how
often and for how long (e.g. twice daily for the first 72 h, then daily until 100% + 7 days), and
the alert threshold per metric that triggers the Rollout Plan's hold/rollback rules (thresholds
cited from `08-knowledge/android/play-vitals-performance.md`, not restated). Include the
fallback if a console's vitals lag (e.g. Crashlytics real-time view) when applicable.

### 7. Rollback Plan

The exact procedure, written to be executed under stress: how to halt the staged rollout in each
store console (numbered steps); whether rollback means halt-only or publishing a previous/fixed
build (Google Play cannot un-ship an installed update — state this and the consequence); the
prior-good version (versionName/versionCode) to re-promote if needed; and a **data-migration
reversibility statement**: every schema/DataStore migration in this release listed as reversible
or one-way, and if one-way, what that means for users who installed the new version ("downgrade
crashes on DB open — rollback protects only not-yet-updated users"). "Revert if problems" is not
a plan and fails validation.

### 8. Store Submission State

A living log, one row per store, appended as states change:

| Store | State (Not submitted / Submitted / In review / Approved / Live at N% / Halted / Rejected) | Date | Notes |
|---|---|---|---|

### 9. Post-Release Actions

Two parts: **Review-response plan** — cadence and tone guideline for responding to user reviews
in the first 14 days, which severities get a response, who responds; **Next-cycle backlog seeds**
— a table of items discovered during this cycle but deferred (ID, description, severity/effort,
where tracked), explicitly addressed to the next-cycle agent that will read this handover during
its P1 discovery.

### 10. Sign-off Log

A living log of human approvals, one row per decision:

| Date | Decision (e.g. Approve upload / Advance to 25% / Hold / Rollback) | Approver | Basis (gate results, metric values) |
|---|---|---|---|

The upload step in the Publish/rollout flow may not be marked done anywhere (here, Store
Submission State, or `docs/factory/PROJECT_STATE.md`) before an "Approve upload" row exists.

## Required Deliverables

1. The handover document at `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md`.
2. The release AAB(s) at the stated path(s), matching the stated checksums.
3. The verification report referenced in Verification Summary, present under `docs/factory/reports/`.
4. The git tag pushed and pointing at the stated commit SHA.

## Validation Rules (mechanically checkable)

| # | Rule |
|---|---|
| V1 | All ten required H2 sections present, in order, non-empty (only `## Rejection Log` and `## Validation Appendix` may appear additionally; Store Submission State and Sign-off Log grow in place) |
| V2 | Version consistency: versionName/versionCode identical across Handover Header, Release Identity, release tag (`release/vX.Y.Z`), release-notes blocks, and the referenced `VERIFICATION_REPORT_v<versionName>.md`; AAB checksum matches the file on disk |
| V3 | Verification Summary lists every release-readiness gate; verdict is PASS, or PASS with waivers where every WAIVED row has an approver; zero unwaived FAIL rows (no "CONDITIONAL") |
| V4 | Every per-locale release-notes block shows `(used/limit)` and used ≤ limit |
| V5 | Rollout Plan defers stage/dwell/threshold numbers to `10-releases/release-workflow.md` stage 4 (does not restate them), names the decision owner and metric source per stage, and its decision rules cover hold, advance, and rollback |
| V6 | No stage in Store Submission State shows a higher rollout % than the highest stage whose advance gate is recorded in the Sign-off Log (no stage advance without the prior stage's gate recorded) |
| V7 | Rollback Plan contains numbered console steps, a prior-good version, and a reversibility statement covering every migration in the release (concrete — fails if it reduces to "revert if problems") |
| V8 | Sign-off Log contains an "Approve upload" row dated on or before any Submitted state in Store Submission State |
| V9 | All referenced files exist (AAB, verification report, changelog); no unresolved references; no placeholder tokens (`{{...}}`), no "TBD"/"TODO" |

## Acceptance Criteria (consumer's bar)

The operator accepts only if all hold; otherwise rejects:

1. **The 2-a.m. test**: if crash rate spikes at any stage, the operator can execute the rollback
   from Section 7 alone — every step, console location, and fallback version is on the page.
2. Every advance/hold decision the operator will face in the next two weeks has a pre-agreed
   numeric rule and a named owner — no judgment calls invented mid-rollout.
3. The artifact is provably the verified one: checksum, tag, and verification report agree.
4. A next-cycle agent reading only this document learns what shipped, how rollout went (via the
   living logs), and what was deliberately left undone.
5. All Validation Rules V1–V9 pass on the consumer's re-run of
   `07-checklists/handover-validation.md`.

## Rejection Protocol

1. The consumer appends a `## Rejection Log` section to the handover file (a permitted append
   alongside `## Validation Appendix` and the living-log sections), one row per reason:

   | Date | Rejected by | Rule / criterion violated | Detail |
   |---|---|---|---|

2. The consumer sets Status to Rejected; nothing is uploaded to any store from a rejected handover.
3. The producer issues `RELEASE_HANDOVER_v<N+1>.md` addressing every logged reason, with
   Supersedes set to vN. The rejected file is never deleted or edited further.
4. Both events are recorded in `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md`.

## Producer & Consumer Workflow

1. **Producer** (prompt 16) drafts with Status: Draft only after the verification report from
   prompt 15 exists with a PASS (or waivable) verdict. Checksums are computed from the actual
   AAB (`Get-FileHash -Algorithm SHA256 release\notespro-3.2.0.aab` on Windows, `sha256sum` on
   POSIX), never copied from CI logs by hand.
2. Producer self-runs `07-checklists/handover-validation.md` (rules V1–V9) and
   `07-checklists/release-readiness.md`.
3. Producer sets Status: Delivered, records delivery in `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md`.
4. **Consumer** (operator) re-runs the gate, verifies the checksum against the file, and walks
   the Rollback Plan mentally (the 2-a.m. test) before accepting.
5. On accept: operator records "Approve upload" in the Sign-off Log, uploads, and drives the
   Rollout Plan — appending to Store Submission State and Sign-off Log at every state change.
   These two living-log sections are the only sections that change after acceptance.
6. After 100% + the monitoring window, the operator closes the release: final log rows, release
   recorded in `docs/factory/reports/` per `09-templates/release-report.md`, and Post-Release
   Actions seeds confirmed in PROJECT_STATE for the next cycle's P1 discovery to pick up.

## Common Failure Modes

| Failure | Symptom downstream | Prevention |
|---|---|---|
| Checksum/tag drift (rebuild after verification) | The shipped binary is not the verified binary | V2 version-consistency check at both delivery and acceptance |
| Rollout advanced on gut feel during a quiet weekend | Vitals breach discovered at 50% instead of 10% | V6 + Sign-off Log: no advance without the prior gate recorded with metric values |
| Rollback plan assumes Play can revert installed updates | Operator "rolls back" and users keep crashing | Section 7 must state halt-only semantics and the prior-good re-promote path |
| One-way DB migration undisclosed | Re-promoting prior version crashes updated users on open | V7 reversibility statement covers every migration |
| Release notes over locale limit | Console truncates or rejects at submit time | V4 `(used/limit)` counts per locale |

## Example Skeleton

```markdown
# RELEASE Handover — NotesPro 3.2.0 (320)

## 1. Handover Header
| Field | Value |
|---|---|
| App | NotesPro (com.example.notespro) |
| Release version | 3.2.0 (320) |
| Handover version | v1 |
| Producer | 01-core/prompts/16-release-preparation.md |
| Date produced | 2026-06-24 |
| Factory version | 1.1.0 |
| Status | Delivered |
| Supersedes | — |
| Stores targeted | Google Play, RuStore |

## 2. Release Identity
| Field | Value |
|---|---|
| Release tag | release/v3.2.0 → 8f3a2c1 |
| AAB | release/notespro-3.2.0.aab — SHA-256 4be1…9c0d |
| Signing | Play App Signing; upload key alias `upload` |
| Build command | ./gradlew :app:bundleRelease |

## 3. Verification Summary
Report: docs/factory/reports/VERIFICATION_REPORT_v3.2.0.md — Verdict: PASS
| Gate | Result | Evidence |
|---|---|---|
| Release build assembles + signs | PASS | CI run #412 |

## 4. Change Summary
### Release notes — en-US (47/500)
`New Material 3 look, dark theme, faster search.`

## 5. Rollout Plan
Stages, dwell times, and thresholds per `10-releases/release-workflow.md` stage 4 (not restated).
| Stage (per release-workflow.md §4) | Advance gate source | Owner | Metric read at |
|---|---|---|---|
| Standard ladder, no deviation | 08-knowledge/android/play-vitals-performance.md | Operator | Play Console → Vitals |

Decision rules: HOLD / ADVANCE / ROLLBACK per release-workflow.md §4 thresholds; owner Operator.

## 6. Monitoring Plan
| Metric | Where | Cadence | Alert threshold |
|---|---|---|---|
| Crash-free users | Play Console → Vitals | 2×/day for 72 h | per play-vitals-performance.md |

## 7. Rollback Plan
1. Play Console → Releases → Production → Halt staged rollout. ...
Prior-good: 3.1.2 (312). Migrations: Room 7→8 is one-way (downgrade crashes on DB open).

## 8. Store Submission State
| Store | State | Date | Notes |
|---|---|---|---|
| Google Play | Submitted | 2026-06-25 | — |

## 9. Post-Release Actions
Review responses: daily triage 14 days; respond to all 1–2★ citing crashes. Backlog seeds:
| ID | Description | Sev/Effort | Tracked in |
|---|---|---|---|
| SEED-01 | Onboarding rework (DES-009) | Medium/L | PROJECT_STATE.md backlog |

## 10. Sign-off Log
| Date | Decision | Approver | Basis |
|---|---|---|---|
| 2026-06-25 | Approve upload | M. Operator | Verification PASS, all gates green |
```
