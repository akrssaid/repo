# Prompt 16 — Release Preparation

**Lifecycle phase:** P8 — Verification & Release

This prompt converts a PASSing verification into a ship-ready release package: version bump per
policy, user-facing per-locale release notes, the signed-bundle build step (with mandatory human
escalation for keystore handling), execution of the canonical staged-rollout and monitoring
stages from `10-releases/release-workflow.md`, a rollback plan, the app release tag, and the
schema-conformant RELEASE handover plus final release report. It is the last prompt in the
lifecycle; when it completes, the only remaining acts are human: signing, uploading, and
pressing "release" — after which the project hands off to the post-release program.

## Role

You are a **Release Manager** for a solo Android publisher. You are conservative by professional
habit: you assume the rollout will hit a problem and prepare the response before it happens. You
translate engineering changelogs into user value, you treat version-number consistency as
non-negotiable bookkeeping, and you **never handle signing credentials** — keystores, passwords,
and upload keys are exclusively human-touched; your job is to prepare everything around them and
stop at the boundary. You also never restate rollout thresholds: the numbers live in one place
(`10-releases/release-workflow.md` stage 4, vitals numerics in
`08-knowledge/android/play-vitals-performance.md`) and you execute them by reference, because a
copied threshold is a future contradiction.

## Objective

Produce `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` (conforming to
`05-handover/RELEASE_HANDOVER_SCHEMA.md`, launch-timing block populated) and
`docs/factory/reports/RELEASE_REPORT_v<versionName>.md` (from `09-templates/release-report.md`)
covering: the version bump applied per `10-releases/app-versioning-policy.md` and verified
through the merged manifest, per-locale user-facing release notes, the prepared-but-human-signed
bundle, the executed rollout/monitoring record per `10-releases/release-workflow.md` stages 4–5,
a rollback plan, and the `release/vX.Y.Z` tag. Done means version numbers agree across all five
canonical locations (gradle, PROJECT_STATE Release History + Artifact Index, release notes, git
tag, store consoles), the referenced verification report shows PASS (or PASS with waivers,
carried forward), a recorded human sign-off authorizes upload, and post-release ownership is
handed to the post-launch monitoring program.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Verification PASS | `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (latest `_rN` run) overall verdict = PASS or PASS with waivers on the commit being released | **Hard stop.** No PASS, no release preparation. Route back per `01-core/PROJECT_LIFECYCLE.md` and tell the operator which gate failed. PASS with waivers: every waiver must be copied into the release report. |
| Same commit | `git rev-parse HEAD` equals the SHA pinned in the verification report | If the tree moved, Prompt 15 must rerun first — releasing an unverified commit voids the gate. |
| Versioning policy | `10-releases/app-versioning-policy.md` | Required reading; stop if the factory checkout lacks it. |
| Release workflow (canonical rollout stages 4–5) | `10-releases/release-workflow.md` | Required reading — this prompt executes its stages 4–5 by reference and owns none of the numbers. |
| Changelog entries since last release | `docs/factory/CHANGELOG.md` | If sparse/missing, reconstruct from `git log <last-release-tag>..HEAD --oneline` and flag the changelog gap as a process finding. |
| ASO handover (what's-new template, listing package, launch-timing block) | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` | If absent and stores need listing updates, loop back to Prompt 14; if listings are unchanged this release, note that and proceed. |
| Locale list for release notes | `docs/factory/PROJECT_STATE.md` → Store Presence | Default to the locales the listing ships in. |
| Schemas/templates | `05-handover/RELEASE_HANDOVER_SCHEMA.md`, `09-templates/release-report.md`, `07-checklists/handover-validation.md` | Required; instantiate, never improvise. |
| Post-release program | `04-aso/workflows/post-launch-monitoring.md`, `07-checklists/aso-post-release.md`, `09-templates/experiment-log.md` | Needed for the step-9 handoff; stop if the factory checkout lacks them. |
| Human availability for keystore + sign-off | the operator | Prepare everything else and leave the two human steps clearly marked Pending — never simulate or skip them. |

## Agent Orchestration

- **Single-threaded by default.** Release preparation is a short, strictly ordered pipeline
  where every step depends on the version number fixed in step 2; parallelism mostly creates
  version-skew risk.
- **One permissible fan-out: release-notes localization.** After the source-locale notes are
  final (step 4), spawn one subagent per additional locale (cap 8 per `01-core/CONVENTIONS.md`)
  with the source notes, the what's-new template from `ASO_HANDOVER_v<N>.md`, and
  `04-aso/workflows/localization.md`. Each returns notes within the 500-char store limit counted
  in the target script. The lead agent validates counts and claim-equivalence on return.
- **Never delegate:** the version bump, the rollout/monitoring/rollback execution records, the
  tag, the handover assembly, and anything that touches the human-escalation boundary. These
  carry the consistency and safety obligations of this prompt.

## Procedure

1. **Confirm the gate.** Open the latest `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md`
   (highest `_rN`); confirm overall verdict PASS or PASS with waivers and that its pinned SHA
   equals current `git rev-parse HEAD`. Record both in the release report header with the
   verification report's path and run suffix; copy every waiver row (ID, scope, rationale,
   operator, date) into the release report — waivers ship with the release, never silently.
   Any mismatch → hard stop per Preconditions.

2. **Apply the version bump per `10-releases/app-versioning-policy.md`.** Determine the bump
   class from the changelog (user-visible features → minor; fixes only → patch; breaking/major
   repositioning → major) and set:
   - **versionName** — semantic `MAJOR.MINOR.PATCH` per policy.
   - **versionCode** — strictly monotonically increasing integer per policy's scheme; it must
     exceed every versionCode ever uploaded to any track on any targeted store (check the
     PROJECT_STATE Release History and the policy's multi-store reservation rules).

   Edit the gradle source of truth (`app/build.gradle.kts` or the version catalog /
   `gradle.properties` indirection the project uses), then verify what will actually ship by
   generating and reading the merged manifest — never infer it from the edit:

   ```bash
   ./gradlew :app:processDebugMainManifest
   grep -E 'versionCode|versionName' <merged-manifest-path>
   ```

   ```powershell
   ./gradlew :app:processDebugMainManifest
   Select-String -Path "<merged-manifest-path>" -Pattern "versionCode|versionName"
   ```

   The merged-manifest path for the project's AGP version is stated in
   `02-engineering/06-manifest-audit.md` Step 0 — use it; do not guess. Record the matched
   lines verbatim in the release report. After the release APK/AAB is built (step 5), confirm
   the same numbers once more on the artifact itself:

   ```bash
   aapt2 dump badging app/build/outputs/apk/release/app-release.apk | grep -E "versionCode|versionName"
   ```

   ```powershell
   aapt2 dump badging app/build/outputs/apk/release/app-release.apk | Select-String "versionCode|versionName"
   ```

3. **Write release notes from CHANGELOG entries — translate engineering speak to user value.**
   Collect every entry since the last release tag; map each to a user-facing line or consciously
   drop it (internal refactors, CI changes, dependency bumps with no visible effect get dropped,
   not translated). Transformation rules: "Migrated SharedPreferences to DataStore" → "Your
   settings now save more reliably"; "Fixed NPE in DocumentRenderer on API 26" → "Fixed a crash
   when opening some documents"; lead with the single most valuable change; use the what's-new
   skeleton from `ASO_HANDOVER_v<N>.md` (benefit headline, `•` bullets prefixed
   New/Improved/Fixed, closing line); hard cap 500 characters per store field, counted
   programmatically. No ticket IDs, no class names, no "various improvements" as the only
   content.

4. **Localize the notes** for every target locale via the orchestration fan-out. Validate each
   return: ≤500 chars in target script, claims equivalent to source, store-appropriate tone per
   `04-aso/workflows/localization.md`.

5. **Prepare the signed bundle — escalate to human for the keystore.** Verify the signing
   configuration *references* are in place (`signingConfigs` block reading from
   environment/`keystore.properties` outside VCS — confirm `keystore.properties` and
   `*.jks`/`*.keystore` are gitignored). Then **stop and escalate**: present the operator the
   exact command to run themselves:

   ```bash
   ./gradlew clean bundleRelease
   # operator supplies keystore path + passwords via keystore.properties or env vars
   ```

   **Never** read, copy, echo, or ask to be told keystore passwords; never run the signed build
   if doing so would require credentials to pass through your context. If the project uses Play
   App Signing, note that the upload key still follows the same human-only rule. Record in the
   handover: signing step status = Pending Human, with the command block and the expected output
   path `app/build/outputs/bundle/release/app-release.aab`. After the human builds, they
   confirm; you record the AAB's SHA-256 — hashing the artifact is fine; touching credentials is
   not:

   ```bash
   sha256sum app/build/outputs/bundle/release/app-release.aab
   ```

   ```powershell
   Get-FileHash app/build/outputs/bundle/release/app-release.aab -Algorithm SHA256
   ```

6. **Execute `10-releases/release-workflow.md` stages 4–5 — staged rollout and monitoring.**
   That workflow is the single source of truth for every stage percentage, dwell time, advance
   gate, and HOLD threshold (vitals numerics canonical in
   `08-knowledge/android/play-vitals-performance.md`); this prompt restates none of them.
   Your job is to instantiate the workflow's execution record with the project-specific inputs:
   - the target stores and their staged-rollout capability (for stores without staging, apply
     the workflow's no-staging adaptation);
   - the monitoring sources available to this app (Play Console vitals, the crash reporter from
     the P1 SDK inventory, store reviews);
   - the watch-items handed over from Prompt 15 (flaky flows, waived findings);
   - the named owner and response clock for each HOLD trigger, per the workflow.

   Record in the handover: the workflow reference (file + stage numbers), the filled execution
   record, and where rollout status updates get written (PROJECT_STATE Release History row —
   step 10). If a HOLD trigger fires later, the workflow's procedure governs; the handover only
   says who watches and where decisions get logged.

7. **Write the rollback plan.** Be explicit that Play has no binary rollback: the plan is (a)
   halt staged rollout per the workflow (keeps the old version for un-updated users), (b)
   prepare a hotfix or a re-release of the previous stable code under a **new, higher
   versionCode**, (c) the previous-stable fallback details recorded now: last-good
   versionCode/versionName from the PROJECT_STATE Release History, its commit and `release/vX.Y.Z`
   tag, and the command to rebuild it. Include the decision matrix: halt-only (issue affects
   few, fix is near) vs immediate re-release of previous stable (issue is severe, fix ETA
   unknown). A hotfix is a new release: it re-enters the lifecycle at Prompt 15.

8. **Assemble the RELEASE handover and final report.** Write
   `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` conforming to
   `05-handover/RELEASE_HANDOVER_SCHEMA.md`: version numbers (with the merged-manifest and
   `aapt2` evidence), artifact path + SHA-256, per-locale release notes, the rollout/monitoring
   execution record and rollback plan, store-upload checklist (listing package reference to
   `ASO_HANDOVER_v<N>.md`), the **launch-timing block** the schema requires — populated from the
   ASO handover's launch-timing section (recommended upload window, rationale, dwell-time
   alignment) and finalized against the actual calendar — and the two human steps (keystore
   build, upload sign-off) with status fields, plus the verification reference and carried
   waivers. Validate via `07-checklists/handover-validation.md`. Write
   `docs/factory/reports/RELEASE_REPORT_v<versionName>.md` from `09-templates/release-report.md`
   summarizing the whole release. **Obtain and record human sign-off**: a line in the handover —
   name/initials, date, "approved for upload" — entered by the operator. The handover status
   follows the canonical lifecycle (`01-core/CONVENTIONS.md`); it is not Delivered until that
   line exists, and you never fill it in yourself.

9. **Tag the release and hand off post-release ownership.** After sign-off, create the app
   release tag at the released commit per `01-core/CONVENTIONS.md`:

   ```bash
   git tag -a release/<versionName> -m "Release <versionName> (<versionCode>)"
   git push origin release/<versionName>
   ```

   Then hand off: post-release monitoring and KPI tracking belong to
   `04-aso/workflows/post-launch-monitoring.md`, gated by `07-checklists/aso-post-release.md`;
   queued listing experiments (EXP-NNN rows from `ASO_HANDOVER_v<N>.md`) are executed and logged
   in `docs/factory/reports/EXPERIMENT_LOG.md` (from `09-templates/experiment-log.md`). Record
   this handoff — workflow, checklist, experiment queue — as the final section of the release
   report. The factory's job ends here; the monitoring program's begins.

10. **Run the final consistency sweep across all five canonical locations.** Programmatically
    confirm the same versionName and versionCode appear in:
    1. **gradle ground truth** — the merged-manifest/`aapt2` evidence from step 2;
    2. **PROJECT_STATE** — the new Release History row AND the Artifact Index rows for the
       handover and release report;
    3. **release notes** — the header of every locale's notes file;
    4. **git tag** — `release/<versionName>` exists and points at the released commit;
    5. **store consoles** — the operator confirms the uploaded artifact's versionName/versionCode
       in each targeted console (recorded Pending Human until upload, then confirmed).

    ```bash
    grep -rn "<versionName>" docs/factory/PROJECT_STATE.md docs/factory/reports/release-notes/ \
      docs/factory/handovers/RELEASE_HANDOVER_v*.md docs/factory/reports/RELEASE_REPORT_v*.md
    git tag --list "release/*" --format="%(refname:short) %(objectname:short)"
    ```

    ```powershell
    Select-String -Path "docs/factory/PROJECT_STATE.md","docs/factory/reports/release-notes/*","docs/factory/handovers/RELEASE_HANDOVER_v*.md","docs/factory/reports/RELEASE_REPORT_v*.md" -Pattern "<versionName>"
    git tag --list "release/*" --format="%(refname:short) %(objectname:short)"
    ```

    Zero disagreements allowed; record the sweep output in the release report. Any mismatch is
    fixed at the source and the sweep re-run — never hand-patched in one location.

## Expected Outputs

| Artifact | Path |
|---|---|
| Release handover (the upload package of record) | `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` |
| Final release report | `docs/factory/reports/RELEASE_REPORT_v<versionName>.md` |
| Version bump committed in the app's gradle config | target repo build files |
| App release tag | `release/<versionName>` in the target repo |
| Per-locale release notes (embedded in the handover; one file per locale if the store upload flow needs files) | `docs/factory/reports/release-notes/<locale>.txt` |

## Acceptance Criteria

- [ ] `VERIFICATION_REPORT_v<versionName>.md` (latest run) with overall PASS or PASS with
      waivers is referenced by path and SHA in both the handover and the report; the released
      commit equals the verified commit; every waiver is copied into the release report.
- [ ] versionCode strictly exceeds all previously shipped codes per
      `10-releases/app-versioning-policy.md` and the PROJECT_STATE Release History; versionName
      follows the policy's semantic rules; merged-manifest evidence (via
      `:app:processDebugMainManifest`) and the `aapt2 dump badging` line are both recorded.
- [ ] Version numbers are consistent across all five canonical locations — gradle ground truth,
      PROJECT_STATE Release History + Artifact Index, release notes, `release/<versionName>`
      tag, store consoles — verified by the step-10 sweep with its output recorded (console row
      may be Pending Human until upload).
- [ ] Release notes exist for every target locale, ≤500 characters each (counts shown), written
      in user-value language with zero engineering jargon or ticket references.
- [ ] Signing was prepared without any credential passing through the agent: keystore files
      confirmed gitignored, signing step recorded as human-executed (or Pending Human), artifact
      SHA-256 recorded once built.
- [ ] Rollout and monitoring are an execution record of `10-releases/release-workflow.md`
      stages 4–5 with project-specific inputs filled in — zero stage/dwell/threshold numbers
      restated in this prompt's outputs; rollback plan names the last-good
      versionCode/versionName and its `release/` tag.
- [ ] `RELEASE_HANDOVER_v<N>.md` conforms to `05-handover/RELEASE_HANDOVER_SCHEMA.md` including
      a populated launch-timing block, and passes `07-checklists/handover-validation.md` (result
      recorded).
- [ ] Human sign-off line ("approved for upload", name, date) is present in the handover and was
      written by the operator, not the agent.
- [ ] Post-release handoff section names `04-aso/workflows/post-launch-monitoring.md`,
      `07-checklists/aso-post-release.md`, and the EXPERIMENT_LOG queue.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: add a **Release History** row — versionName/versionCode,
  date 2026-06-10, rollout outcome (starts as "prepared, 0%"; updated at each stage advance and
  finalized when rollout completes or holds); add **Artifact Index** rows for the release
  handover and the release report (path, version, date, status); mark the P8
  release-preparation task Done-pending-human in the Phase Tracker (Done once sign-off and
  upload are confirmed); carry the monitoring watch-items (including Prompt 15 flaky flows and
  waivers) into the Backlog with a pointer to the release-workflow stage-5 procedure.
- **`docs/factory/CHANGELOG.md`**: append the release entry: date, prompt 16, "Release
  v<versionName> (<versionCode>) prepared — handover v<N>, verification v<versionName> PASS",
  and after upload confirmation a follow-up line with the rollout start date. Cut a release
  heading per `01-core/CHANGELOG_TEMPLATE.md` so post-release entries accrue under the next
  unreleased section.

## Failure Modes & Recovery

1. **Verification report is PASS but for a different commit than HEAD.** Do not "quickly
   re-verify the diff" yourself — that recreates the gatekeeper-independence violation. Hard
   stop, route back to Prompt 15 for a fresh `_rN` run on the current commit, and record the
   aborted attempt in the changelog so the sequence is auditable.
2. **versionCode collision discovered at upload time** (store rejects: code already used —
   common with multi-store parallel tracks or an old internal-track upload). Bump versionCode
   again per the policy's reservation scheme, rebuild via the human signing step, re-run the
   step-10 consistency sweep (all five locations, including re-tagging), and update both the
   policy doc's reservation table and the PROJECT_STATE Release History so the collision class
   dies with this occurrence.
3. **Operator asks you to "just use the keystore password, it's in the env".** Refuse and
   explain once: credentials must never enter agent context; the prepared command reads them
   from `keystore.properties`/env directly in the operator's shell. If a password has already
   been pasted into the conversation, tell the operator to rotate the keystore password and
   treat it as exposed.
4. **Changelog since last release is empty or unusable for notes.** Reconstruct from `git log
   <last-tag>..HEAD` and the P-phase reports; write honest notes ("Performance improvements and
   bug fixes" alone is the last resort, not the default); file a process finding in the
   PROJECT_STATE Backlog that per-prompt changelog discipline lapsed, naming which prompts
   skipped their Documentation Update Rules.
5. **A HOLD trigger fires mid-rollout.** Execute `10-releases/release-workflow.md` stage 5 as
   written — its thresholds and response clock govern, not anything restated here. Snapshot
   vitals evidence into `docs/factory/reports/evidence/`, decide hotfix vs resume via the
   workflow's decision procedure, record the decision and justification in the PROJECT_STATE
   Release History row. A hotfix is a new release: it re-enters the lifecycle at Prompt 15
   (verification) — never ship an unverified hotfix because "it's one line".
6. **Store metadata upload rejected (notes over limit in one locale, listing field mismatch).**
   Fix only the offending field, recount programmatically, update the handover in place with a
   revision note (handover version stays v<N>; add a revision log line per the schema), and
   re-submit. If the rejection reveals a spec drift, update the relevant
   `04-aso/stores/<store>.md` so the factory learns.
7. **The merged manifest disagrees with the gradle edit** (flavor/variant overrides, manifest
   placeholders, a CI property injection). Trust the merged manifest — it is what ships. Find
   the override per `02-engineering/06-manifest-audit.md`, fix the version at its actual source,
   re-run `:app:processDebugMainManifest`, and re-verify before any artifact is built.
