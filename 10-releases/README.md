# 10-releases — Release Management

This directory defines how TARGET apps get shipped (the operational release workflow and the
versioning policy every release must follow) and where the factory tracks changes to itself
(`FRAMEWORK_CHANGELOG.md`). Everything here is consumed during lifecycle phase **P8 —
Verification & Release**; nothing in this directory applies before a verification report exists.

## What lives here

| File | Scope | Purpose |
|---|---|---|
| `10-releases/release-workflow.md` | Target apps | The end-to-end release procedure: pre-flight, build & sign, store submission, staged rollout, monitoring, rollback, close-out. |
| `10-releases/app-versioning-policy.md` | Target apps | The factory standard for `versionName`/`versionCode`, multi-store strategy, bump procedure, and consistency rules. |
| `10-releases/FRAMEWORK_CHANGELOG.md` | The factory itself | Keep-a-Changelog history of this repository. Every change to the factory lands here. |

## How this connects to the rest of the factory

```
P8 — Verification & Release
  ├─ 01-core/prompts/15-final-verification.md  → produces VERIFICATION_REPORT_v<versionName>.md (must be PASS)
  ├─ 01-core/prompts/16-release-preparation.md → produces RELEASE_HANDOVER + release notes
  ├─ 07-checklists/release-readiness.md        → gate run during pre-flight
  ├─ 05-handover/RELEASE_HANDOVER_SCHEMA.md    → shape of the handover the workflow consumes
  ├─ 10-releases/app-versioning-policy.md      → version decided here, before any build
  ├─ 10-releases/release-workflow.md           → the procedure itself (rollout-gate single source)
  └─ 09-templates/release-report.md            → close-out artifact in docs/factory/reports/

post-release loop
  ├─ 04-aso/workflows/post-launch-monitoring.md → ongoing KPI watch after the 72h window closes
  └─ 07-checklists/aso-post-release.md          → post-release gate (store presence + vitals + KPIs)
```

- **Lifecycle**: `01-core/PROJECT_LIFECYCLE.md` defines P8. Prompt 15 ends with a verification
  verdict; prompt 16 prepares the release package. `release-workflow.md` is what executes that
  package against real store consoles.
- **Handover**: the release is driven from `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md`,
  which must conform to `05-handover/RELEASE_HANDOVER_SCHEMA.md` and pass
  `07-checklists/handover-validation.md` before stage 1 of the workflow begins.
- **Readiness gate**: `07-checklists/release-readiness.md` is run verbatim in pre-flight. Any
  unchecked item blocks the release — no exceptions, no "we'll fix it in the next build".
- **Rollout gates — single source**: `release-workflow.md` **stage 4** is the SINGLE SOURCE for
  every rollout stage, dwell, and threshold number (10→25→50→100%, minimum 24h dwell per stage,
  HOLD on crash baseline +0.2pp or ANR ≥0.40%). Prompt 16 and the RELEASE handover schema delete
  their own tables and defer to it. No rollout number is restated anywhere else in this directory.
- **Vitals numbers — single source**: the underlying Play bad-behavior thresholds (1.09% crash /
  0.47% ANR) and startup-time bands live in `08-knowledge/android/play-vitals-performance.md`.
  Release docs cite that file for the numeric basis; they do not redefine it.
- **Post-release gate & monitoring loop**: once the 72h watch closes, the
  `07-checklists/aso-post-release.md` gate confirms store presence, vitals, and KPI capture, and
  `04-aso/workflows/post-launch-monitoring.md` runs the continuing KPI/experiment loop.
- **Store knowledge**: track mechanics and per-store quirks come from `04-aso/stores/*` and
  `08-knowledge/stores/store-comparison.md` (source of the Play-first canary doctrine).
- **Conventions**: all branch/commit/tag formats, ID registry, scales, verification tiers, and
  canonical paths cited here are defined once in `01-core/CONVENTIONS.md`; this directory cites
  that file rather than restating its rules.

## Release doctrine (non-negotiable)

1. **No release without verification PASS.** The verification report from prompt 15
   (`VERIFICATION_REPORT_v<versionName>.md`) must read **PASS** or **PASS with waivers** where
   every waiver is named and accepted in the handover. The verdict vocabulary is exactly **PASS /
   PASS with waivers / FAIL** per `01-core/CONVENTIONS.md` — the word "CONDITIONAL" is abolished.
   A FAIL verdict loops back to P8 start; an unwaived gap is a FAIL.
2. **Staged rollouts, always.** Production releases on Google Play start at 10% and earn each
   promotion. 100%-on-day-one is reserved for emergency forward-fixes only, and even those start
   at 50% unless users are actively losing data.
3. **Rollback plan before upload.** The RELEASE_HANDOVER must contain a written rollback/
   forward-fix plan — including the data-migration reversibility check — before any artifact is
   uploaded anywhere. "We'll figure it out if it breaks" is a blocked release.
4. **Human owns the upload button.** The agent prepares everything: builds, checksums, release
   notes, console-ready text. The human performs keystore signing custody, console uploads, track
   promotions, and rollout-percentage changes. The agent never holds signing credentials or store
   console sessions. The swimlane in `release-workflow.md` makes the split explicit.

## Where per-project release artifacts live

In the TARGET app repo, not here:

| Artifact | Path |
|---|---|
| Release handover | `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` |
| Verification report | `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (from prompt 15; re-runs append `_r2`, `_r3`) |
| Release report (close-out) | `docs/factory/reports/RELEASE_REPORT_v<versionName>.md` (instantiated from `09-templates/release-report.md`) |
| Project memory + changelog | `docs/factory/PROJECT_STATE.md` (Release History + Artifact Index), `docs/factory/CHANGELOG.md` |

## Reading order

1. New to the factory's release process → this README, then `release-workflow.md` end to end.
2. Deciding a version number → `app-versioning-policy.md` only; it is self-contained.
3. Changing the factory itself → append to `FRAMEWORK_CHANGELOG.md` in the same commit, and
   follow the version bump rules cross-referenced in `VERSION.md`.
