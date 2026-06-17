# Checklists — Gate Doctrine

This directory contains the gate checklists of the android-product-factory. A checklist here is
not a suggestion or a reminder list — it is a **gate**: a phase or handover does not exit, and its
artifacts are not consumed downstream, until the relevant checklist has been run and passed. Every
item in every checklist is objectively verifiable: a command to run, an artifact to confirm
exists, or a value to compare against a stated threshold. If an item cannot be verified that way,
it does not belong in a checklist.

Scales, naming, ID formats, canonical paths, and verification tiers referenced throughout these
checklists are defined once in `01-core/CONVENTIONS.md`; checklists cite it rather than restating
the rules.

## Doctrine

1. **Gates, not suggestions.** A checklist run is mandatory at the lifecycle point that invokes
   it. Skipping a gate is a process violation and must be recorded in the target app's
   `docs/factory/CHANGELOG.md` with a reason.
2. **Pass condition = 100% of non-N/A items checked.** There is no partial pass. A single failed
   item means the verdict is FAIL and the phase does not exit until the item is fixed and the
   checklist re-run (re-run only the failed group plus any group it could affect).
3. **N/A requires written justification.** An item may be marked N/A only with a one-line written
   justification next to the item (e.g. "N/A — app has no native libraries, verified via
   `Get-ChildItem -Recurse -Filter *.so`"). An unjustified N/A counts as a FAIL.
4. **Run by an agent, signed by evidence.** Claude agents run checklists; every checked item must
   cite its evidence (command output, file path, screenshot path). "Looks fine" is not evidence.
5. **Checklists are versioned with the factory.** If a checklist item is wrong or outdated, fix it
   in this repo and log the change in `10-releases/FRAMEWORK_CHANGELOG.md` — never silently skip
   it on a project.

## Where each checklist sits in the lifecycle

| Checklist | Gate for | Invoked by | Blocks |
|---|---|---|---|
| `07-checklists/engineering-modernization.md` | P4 (Modernization) exit | `02-engineering/migrations/*` completion, `01-core/PROJECT_LIFECYCLE.md` P4 exit | Entry to P5 |
| `07-checklists/design-modernization.md` | P6 (Redesign Implementation) exit | `01-core/prompts/09-ui-implementation.md` (its acceptance criteria require this checklist to pass) | Entry to P7 |
| `07-checklists/accessibility.md` | Within P6 and P8 | `03-design/09-accessibility-review.md`, `01-core/prompts/15-final-verification.md` | P6 exit and P8 release sign-off |
| `07-checklists/aso-launch.md` | P7 (ASO & Store Assets) exit | `01-core/prompts/14-store-assets.md`; re-run before any listing publish | Listing publish, entry to P8 |
| `07-checklists/aso-post-release.md` | Post-release (after a release goes live) | `04-aso/workflows/post-launch-monitoring.md`; re-run on the monitoring cadence | Closing out the release / declaring ASO outcome |
| `07-checklists/release-readiness.md` | P8 (Verification & Release) | `01-core/prompts/15-final-verification.md`, consumed by `16-release-preparation.md` | Store submission |
| `07-checklists/handover-validation.md` | Every handover, any phase | Producer before delivery; consumer before consuming any `docs/factory/handovers/*_HANDOVER_v<N>.md` | Consumption of the handover |

`handover-validation.md` is universal: it runs at minimum once per governed handover type
(DESIGN, REDESIGN_PROPOSAL, IMPLEMENTATION, ASO, RELEASE) and again on every new handover version.

## How to run a checklist

1. Copy the checklist file from this directory into the target app repo as
   `docs/factory/reports/<checklist-name>-YYYY-MM-DD.md` (e.g.
   `docs/factory/reports/engineering-modernization-2026-06-10.md`).
2. Work through every item in the copy. Check `[x]` only after executing the stated verification;
   paste the key evidence (command output line, file path) under or beside the item.
3. Mark N/A items with justification inline. Leave failed items unchecked and list them in the
   Sign-off block.
4. Complete the Sign-off block: date, verdict PASS/FAIL, failed items, run by.
5. Log the run in `docs/factory/CHANGELOG.md`: date, checklist name, verdict, and the report path.
   On FAIL, also log the remediation tickets or actions created.
6. On FAIL: fix, then re-run the failed group(s) in a NEW dated report copy — never edit a
   completed report to flip a verdict.

## Recording results

- Completed checklist copies are permanent project artifacts in `docs/factory/reports/`. Never
  overwrite a prior run; the date suffix keeps history.
- The PASS report for a phase gate must exist before `docs/factory/PROJECT_STATE.md`'s phase
  tracker is advanced to the next phase.
- Auditors reconstruct project history from `docs/factory/reports/` + `docs/factory/CHANGELOG.md`;
  keep both consistent.

## Maintaining these checklists

Checklists encode the tech baseline of June 2026 (compileSdk 36, Kotlin 2.1+, AGP 8.9+, Gradle
8.13+ — see `08-knowledge/android/version-matrix.md`). When the baseline moves, update the
affected items here and record the change in `10-releases/FRAMEWORK_CHANGELOG.md`.
