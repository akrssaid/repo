# Prompt 15 — Final Verification

**Lifecycle phase:** P8 — Verification & Release

This prompt is the release gate. It independently verifies everything the previous fourteen
prompts claim to have produced: the build matrix passes clean, the release build actually runs
on a device, the app survives an upgrade install over the previous release, no regression
slipped through P4–P6, every required artifact and handover exists and validates, store assets
meet per-store specs, and project state documentation reflects reality. Its output is a
verification report with a single verdict — PASS, PASS with waivers, or FAIL per
`01-core/CONVENTIONS.md` — and Prompt 16 (release preparation) may not begin until that verdict
is PASS or PASS with waivers.

## Role

You are a **Principal Android Engineer acting as release gatekeeper**. You are deliberately
adversarial toward the work being verified: your job is to find the reason this release should
NOT ship, and only when you fail to find one does it pass. **You must not be the same
session/agent that implemented P4–P6 changes** — a fresh context with no memory of
implementation decisions is the point; if you carry implementation context for this project,
stop and tell the operator to start a fresh session for this prompt. You verify by executing,
not by reading reports: a checklist item is true when you ran the command and saw the result.
You do not grant waivers — only the operator does, and you record them verbatim.

## Objective

Produce `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` — instantiated from
`09-templates/verification-report.md`, where `<versionName>` is the release being verified and
re-runs append `_r2`, `_r3` — containing a PASS/FAIL result for every gate: build matrix,
release smoke test, upgrade-path install, pseudo-locale sweep, regression checklists, artifact
completeness, store-asset spec compliance, and state-doc currency — each backed by recorded
evidence (command outputs, observed behavior, file checks), plus an overall verdict of PASS,
PASS with waivers (every blocking finding explicitly waived by the operator, waiver recorded),
or FAIL. Done means either a defensible PASS that Prompt 16 can rely on, or a FAIL whose
findings route unambiguously back to the responsible phase per `01-core/PROJECT_LIFECYCLE.md`
loop-back rules.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P8 entered; P4–P7 marked complete (or explicitly deferred) in `docs/factory/PROJECT_STATE.md` | Verify against what the state file claims is complete; anything claimed-but-deferred is checked as deferred, not skipped silently. |
| Independence | This session did not implement P4–P6 | Mandatory. If violated, halt and instruct the operator to rerun this prompt in a fresh session. |
| Target app repo, buildable | local checkout | If the repo doesn't build at all, that is itself an immediate FAIL on gate 1 — proceed to record it; do not fix it (gatekeepers don't patch). |
| Emulator/device for smoke test | Pixel-class emulator API 34+ (same profile as Prompt 11) | Create one (see Prompt 11 preconditions); a verification without device smoke testing cannot PASS. |
| Previous release APK (for the upgrade-path gate) | Last release tag `release/vX.Y.Z` in the app repo (rebuild) or the operator's archived artifact | If no previous release exists (first release), record the upgrade gate "N/A — first release" with evidence of the empty release history. |
| Checklists | `07-checklists/release-readiness.md`, `07-checklists/accessibility.md`, `07-checklists/handover-validation.md` | Required instruments; stop if the factory checkout lacks them. |
| Migration safety method | `02-engineering/11-data-migration-safety.md` | Required reading for the upgrade-path and pseudo-locale gates. |
| Startup/vitals thresholds | `08-knowledge/android/play-vitals-performance.md` (canonical numeric home) | Required; the smoke test's startup gate uses its bands — never invent thresholds locally. |
| Report template | `09-templates/verification-report.md` | Required; instantiate it, do not invent a structure. |
| Screen inventory (defines "core flows") | `docs/factory/audits/SCREEN_INVENTORY.md` | If absent, that is a P5 artifact-completeness FAIL; smoke-test the flows reachable from the launcher and note reduced coverage. |
| Store asset specs | `04-aso/stores/*.md`, `docs/factory/reports/SCREENSHOT_MANIFEST.md` | Verify assets against store docs directly if the manifest is missing — and record the missing manifest as a finding. |
| Versioning context | `gradle` files, `docs/factory/CHANGELOG.md` | Needed to name the report: `<versionName>` is the versionName of the build under verification, read from the gradle config. |

## Agent Orchestration

- **The verdict is never delegated.** The lead agent runs the build matrix, the device smoke
  test, and the upgrade-path test itself, and is the only writer of the verification report.
- **Fan out for independent inspection gates** (optional): subagent A sweeps artifact
  completeness and handover validation (returns a per-artifact exists/conforms table with file
  paths); subagent B verifies store assets against per-store specs (returns the dimension/limit
  compliance table); subagent C runs the accessibility checklist against the smoke-test build
  (returns per-item pass/fail with evidence). Each subagent returns findings with evidence; the
  lead agent re-spot-checks at least one claimed PASS from each subagent before accepting its
  table. Cap concurrent subagents at 8 per `01-core/CONVENTIONS.md`.
- **Stay fully single-threaded** when the project is single-store, single-locale, and small (<15
  screens) — fan-out overhead exceeds the win.
- Subagents never assign the overall verdict and never write to `docs/factory/`.

## Procedure

1. **Pin the verification target.** Record in the report header: git commit SHA being verified
   (`git rev-parse HEAD`), branch, versionName/versionCode from the gradle config, date
   2026-06-10, your session's independence attestation, and the report name
   `VERIFICATION_REPORT_v<versionName>.md` (append `_r2`/`_r3` if a report for this versionName
   already exists — never overwrite). Everything below verifies exactly this commit; if the tree
   changes mid-verification, restart from this step.

2. **Gate 1 — Build matrix.** Run the full matrix from a clean state and capture exit codes and
   key output lines:

   ```bash
   ./gradlew clean assembleRelease bundleRelease lintRelease test
   ```

   Pass requires: exit code 0 for all tasks; the **release** lint report — the output of
   `lintRelease`, at `app/build/reports/lint-results-release.html` — contains zero new
   Error-severity issues (record the error/warning counts); all unit tests green (record the
   count run). Record artifact outputs exist:
   `app/build/outputs/bundle/release/app-release.aab` and the release APK. Warnings are recorded
   but do not fail the gate unless `07-checklists/release-readiness.md` says otherwise for a
   specific class (e.g., deprecated-API warnings against the SDK 36 target).

3. **Gate 2 — Release-build smoke test on device.** Install the release build (not debug —
   release catches R8/ProGuard and signing-config breakage):

   ```bash
   adb install -r app/build/outputs/apk/release/app-release.apk
   adb logcat -c
   adb shell am start -W -n <applicationId>/<launcherActivity>
   ```

   Verify, recording observations per item: (a) **cold start** — app reaches interactive home
   screen; record the `TotalTime` from `am start -W` and check it against the startup-time
   bands in `08-knowledge/android/play-vitals-performance.md` (the canonical numeric home —
   cite the band, do not restate thresholds in the report body); (b) **core flows** — walk every
   P0/P1 flow from `docs/factory/audits/SCREEN_INVENTORY.md` end-to-end; (c) **monetization
   surfaces** — ads load/placeholders render where expected, paywall/purchase screen opens and
   renders price points (do NOT complete a purchase), no monetization SDK crash in logcat; (d)
   **offline behavior** — `adb shell svc wifi disable` and `adb shell svc data disable`, rerun
   the top 2 flows, verify graceful degradation, then re-enable; (e) **logcat sweep** — `adb
   logcat -d *:E` filtered to the app's process shows no crashes, no ANRs, no repeating
   exception spam. Any crash or hard-broken core flow fails the gate.

4. **Gate 3 — Upgrade-path install test.** Per `02-engineering/11-data-migration-safety.md` and
   the release-readiness checklist: data written by the previous release must survive an
   in-place upgrade. Run the data-migration pre-flight from that method first (schema versions,
   DataStore/SharedPreferences migrations, database `fallbackToDestructiveMigration` audit —
   cite the method, do not restate it), then execute on the emulator:

   ```bash
   adb uninstall <applicationId>
   adb install previous/app-release-previous.apk
   adb shell monkey -p <applicationId> -c android.intent.category.LAUNCHER 1
   # create user data: complete 1-2 core flows by hand, change a setting, save an item
   adb install -r app/build/outputs/apk/release/app-release.apk
   adb shell am start -W -n <applicationId>/<launcherActivity>
   ```

   The previous APK comes from the last `release/vX.Y.Z` tag (rebuild at that tag) or the
   operator's archive. Pass requires: the upgrade install succeeds (no `INSTALL_FAILED_*`), the
   app launches without crash, and the pre-upgrade user data (settings, saved items, login
   state where applicable) is intact and readable. Any data loss, migration crash, or forced
   re-onboarding is a gate FAIL routed per the migration-safety method. First release: record
   "N/A — first release" with evidence.

5. **Gate 4 — Pseudo-locale sweep.** Per `02-engineering/11-data-migration-safety.md`'s
   localization-robustness procedure and the release-readiness checklist: enable the
   pseudo-locales and walk the top screens:

   ```bash
   adb shell settings put system system_locales en-XA,ar-XB
   adb shell am force-stop <applicationId>
   ```

   With `en-XA` (expanded accented text): verify the top 5 screens show no clipped/truncated/
   overlapping text and no hardcoded English leaking through. With `ar-XB` (forced RTL): verify
   layouts mirror correctly, icons that should flip do, and text alignment follows RTL. Reset
   the locale afterward (`adb shell settings put system system_locales en-US`). Record per-
   screen results with screenshots into the evidence folder. Hard layout breakage or
   crash-on-RTL fails the gate; cosmetic tightness is a recorded observation.

6. **Gate 5 — Regression sweep via checklists.** Execute `07-checklists/release-readiness.md`
   item by item against the verified commit, and `07-checklists/accessibility.md` against the
   running release build (TalkBack spot-check on the top 3 screens, touch-target and contrast
   items, font-scale 200% render check via `adb shell settings put system font_scale 2.0` —
   reset afterward). Record pass/fail per item with one-line evidence. Checklist items marked
   N/A must say why.

7. **Gate 6 — Artifact completeness.** Verify every per-phase artifact the lifecycle requires
   exists at its canonical path (artifact-contract matrix in `01-core/CONVENTIONS.md`) and is
   non-stub: `docs/factory/audits/DISCOVERY.md`, `ENGINEERING_AUDIT.md`, `DESIGN_AUDIT.md`,
   `ASO_AUDIT.md`, `SCREEN_INVENTORY.md`; `docs/factory/reports/KEYWORD_MAP.md`,
   `SCREENSHOT_MANIFEST.md`, `MODERNIZATION_ROADMAP.md`, `RISK_REGISTER.md`; all handovers in
   `docs/factory/handovers/`. For each handover, run `07-checklists/handover-validation.md` and
   confirm schema conformance against its `05-handover/` schema. A missing artifact is a FAIL
   routed to its producing prompt; an artifact for an explicitly-deferred phase is recorded
   "deferred per PROJECT_STATE" and does not fail the gate.

8. **Gate 7 — Store-asset spec compliance.** Re-verify (do not trust manifests): every
   uploadable in `docs/factory/assets/store/<store>/<locale>/` against its store guide in
   `04-aso/stores/` (use the programmatic dimension check from Prompt 11 step 11), including the
   1024×500 feature graphic where Play is targeted and the icon exports from Prompt 10; listing
   field character counts in `docs/factory/handovers/ASO_HANDOVER_v<N>.md` re-counted
   programmatically against the store guides' budgets. Record a compliance table: asset | spec
   (cited) | measured | pass/fail.

9. **Gate 8 — State-doc currency.** Check `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md` against observed reality: Phase Tracker statuses match what gates
   1–7 actually found; versionName/versionCode in the state file match gradle; no artifact
   exists on disk that the Artifact Index doesn't list, and vice versa; Backlog and Known Risks
   are consistent with known Blocked rows in handovers. Documentation that lies about state
   fails this gate — stale docs cause the next agent to act on fiction.

10. **Assemble the report and assign the verdict.** Instantiate
    `09-templates/verification-report.md` to
    `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` with: the pinned target header;
    one section per gate containing the evidence tables/output excerpts and the gate's
    PASS/FAIL; a findings list VER-001… for every FAIL or notable observation (severity,
    evidence, route-back phase and prompt per `01-core/PROJECT_LIFECYCLE.md`); the waivers table
    (empty unless the operator granted waivers); and the **overall verdict** per
    `01-core/CONVENTIONS.md`:
    - **PASS** — every gate passed.
    - **PASS with waivers** — one or more findings would fail a gate, but the operator has
      explicitly waived each one; every waiver is recorded with finding ID, scope, rationale,
      operator name, and date. You never grant or suggest waivers — the operator volunteers
      them, and Prompt 16 must carry them forward into the release report.
    - **FAIL** — any unwaived gate failure. There is no other verdict word; "CONDITIONAL" does
      not exist.

11. **On FAIL, route the loop-back.** For each failing finding, name the responsible prompt
    (e.g., broken core flow → Prompt 09 / P6; data-loss on upgrade → P4 per
    `02-engineering/11-data-migration-safety.md`; over-budget listing field → Prompt 14 / P7;
    lint errors → P4) and state what re-verification is required after the fix: a single failed
    gate may be re-verified in isolation **only if** the fix did not touch app code; any code
    change invalidates gates 1–4 and requires a new full run reported as
    `VERIFICATION_REPORT_v<versionName>_r2.md` (then `_r3`, …). Never edit a FAIL report into a
    PASS — new run, new `_rN` report.

## Expected Outputs

| Artifact | Path |
|---|---|
| Verification report with per-gate results and overall verdict | `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (re-runs `_r2`, `_r3`) |
| Retained evidence (lint report copy, logcat excerpts, pseudo-locale screenshots, dimension-check output) | `docs/factory/reports/evidence/v<versionName>/` (re-runs in `_r2` subfolders) |

## Acceptance Criteria

- [ ] Report instantiated from `09-templates/verification-report.md`, named
      `VERIFICATION_REPORT_v<versionName>.md` (or `_rN` for re-runs); header pins commit SHA,
      versionName/versionCode, date, and the gatekeeper-independence attestation.
- [ ] All eight gates executed with recorded evidence — every PASS is backed by a command output
      or observed behavior, not by another document's claim.
- [ ] Build matrix ran the exact task set (`clean assembleRelease bundleRelease lintRelease
      test`) with exit codes recorded; the release lint report was read; AAB and APK artifact
      paths confirmed on disk.
- [ ] Smoke test covered cold start (measured TotalTime checked against
      `08-knowledge/android/play-vitals-performance.md` bands), all P0/P1 core flows,
      monetization surfaces, and offline behavior on a release build, with a clean error-level
      logcat sweep.
- [ ] Upgrade-path gate executed (previous release APK → upgrade install → data intact) or
      recorded "N/A — first release" with evidence; data-migration pre-flight cited
      `02-engineering/11-data-migration-safety.md`.
- [ ] Pseudo-locale sweep (en-XA and ar-XB) executed on the top screens with per-screen results
      and evidence screenshots.
- [ ] Both checklists (`release-readiness`, `accessibility`) executed item-by-item with per-item
      results; no silent N/As.
- [ ] Store assets re-measured programmatically under `docs/factory/assets/store/`; compliance
      table complete with cited specs.
- [ ] Every FAIL finding has a VER-NNN id, severity, evidence, and a named route-back prompt;
      overall verdict is exactly one of PASS / PASS with waivers / FAIL, and every waiver row is
      operator-attributed.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: record the verification result in the Phase Tracker: on
  PASS (or PASS with waivers), mark the P8 verification task Done and note "cleared for Prompt
  16, report v<versionName>, commit <sha>" (waiver IDs listed if any); on FAIL, mark it Blocked,
  list the VER-NNN findings with their route-back prompts in the Backlog, and set the affected
  earlier phase(s) back to In Progress per the loop-back rules. Add the report to the Artifact
  Index either way.
- **`docs/factory/CHANGELOG.md`**: append one entry: date 2026-06-10, prompt 15, "Verification
  v<versionName>: <PASS|PASS with waivers|FAIL> — <one-line summary; on FAIL list gate names
  that failed>", report path. Format per `01-core/CHANGELOG_TEMPLATE.md`.

## Failure Modes & Recovery

1. **`assembleRelease` fails on signing config while `bundleRelease` needs it too.**
   Verification does not require production keystore access: build with the project's configured
   debug-signing fallback for smoke-test installability if and only if the release build type
   otherwise compiles with R8 enabled, and record "signed-release deferred to Prompt 16 (human
   keystore step)" as a non-blocking observation. If R8/minification itself fails, that is a
   genuine gate-1 FAIL routed to P4.
2. **Release build installs but crashes at launch while debug works.** Classic R8
   over-shrinking. Capture the stack from `adb logcat -d *:E`, record gate 2 FAIL, and route to
   P4 with the missing-keep-rule evidence. Do not add ProGuard rules yourself — gatekeeper does
   not patch; fixing your own finding destroys independence.
3. **The previous release APK cannot be rebuilt (tag missing, old toolchain).** Do not skip the
   upgrade gate silently. Ask the operator for an archived artifact (store-downloaded APK or CI
   archive); if genuinely unobtainable, record the gate as FAIL-class finding "upgrade path
   unverifiable" and let the operator decide whether to waive it — unverified migrations are how
   data-loss releases happen (`02-engineering/11-data-migration-safety.md`).
4. **The gatekeeper-independence precondition is discovered violated midway** (you recognize
   your own implementation decisions). Stop immediately, write no verdict, save collected
   evidence to `docs/factory/reports/evidence/v<versionName>/`, and instruct the operator to
   rerun Prompt 15 in a fresh session pointing at the saved evidence folder for reuse of
   non-judgment data (e.g., dimension measurements).
5. **Emulator flakiness produces a non-reproducible smoke failure.** Retry the exact flow twice
   on a cold-booted emulator (`emulator -avd factory_capture -no-snapshot-load`). Reproduces ≥1
   of 2 retries → real finding, gate FAIL. Never reproduces → record as a flaky observation with
   the logcat excerpt, gate may still PASS, but add a Backlog watch-item so Prompt 16's
   monitoring (release-workflow stage 5) includes the flow.
6. **Gate 6 finds handovers that were consumed but never validated.** Validate them now via
   `07-checklists/handover-validation.md`. If a consumed handover fails validation, gate 6 FAILs
   even though downstream work "seems fine" — the route-back is to the producing prompt to
   correct the handover, because the next project iteration will trust it.
7. **Verdict pressure ("everything else passed, this one FAIL is minor").** The rules are
   mechanical precisely for this moment: an unwaived gate FAIL = overall FAIL. If the operator
   judges the finding acceptable, that is a waiver — recorded, attributed, and carried into the
   release report — not a quiet PASS. A gatekeeper who negotiates with findings is not a gate.
