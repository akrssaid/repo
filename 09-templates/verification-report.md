# Template — Verification Report

This template produces the permanent record of the P8 release-readiness verification for one
release candidate: the overall verdict, the result of every gate run against the actual release
build, any waivers, where each failure routes for repair, and the artifacts checked. Its existence
with verdict PASS (or PASS with waivers) is a precondition for `01-core/prompts/16-release-preparation.md`
— nothing ships on a FAIL. The verifier executes `07-checklists/release-readiness.md` and records
the outcome here; this report is the evidence that the gate was actually run on the build being
shipped, not on an earlier or debug build.

## Usage

- **Instantiated by:** the verification agent running `01-core/prompts/15-final-verification.md`,
  executing `07-checklists/release-readiness.md` and `07-checklists/accessibility.md` against the
  release candidate. **The verifier must NOT be the session that implemented the change** — a
  fresh-context verification is the point; record the verifier identity in the header.
- **Phase:** P8 — Verification & Release.
- **Instance path:** `docs/factory/reports/VERIFICATION_REPORT_v{{VERSION_NAME}}.md` in the TARGET
  app repo — one per release. **Re-runs after a FAIL append a suffix:** `_r2`, `_r3`
  (e.g. `VERIFICATION_REPORT_v4.0.0_r2.md`); a rebuilt artifact invalidates the prior report.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy the body below the
  marker, fill all tokens, delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / applicationId | Solar Camera Pro / com.solarcamera.pro |
| `{{VERSION_NAME}}` / `{{VERSION_CODE}}` | versionName / versionCode of the candidate (per `10-releases/app-versioning-policy.md`) | 4.0.0 / 40000 |
| `{{BUILD_COMMIT}}` | Git SHA the release candidate was built from | 9f3c1a7e…(40 chars) |
| `{{VERIFIER}}` | Verifier identity — agent role/session, NOT the implementing session | Verification agent (fresh session, owner-supervised) |
| `{{IMPLEMENTING_SESSION}}` | The session that did the work, named to prove separation | P6/P9 implementation session 2026-06-08 |
| `{{VERIFY_DATE}}` | Date verification completed, ISO 8601 | 2026-06-10 |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{RERUN_LABEL}}` | "" for the first run, "_r2"/"_r3" for re-runs (mirror the filename suffix) | _r2 |
| `{{OVERALL_VERDICT}}` | PASS / PASS with waivers / FAIL | PASS with waivers |
| `{{VERDICT_SUMMARY}}` | One-paragraph justification of the verdict | (prose) |
| `{{GATE_RESULT_ROWS}}` | Rows for the Gate Results table | (table rows) |
| `{{WAIVER_ROWS}}` | Rows for the Waivers log (or "None") | (table rows) |
| `{{FAILURE_ROUTING_ROWS}}` | Rows for the Failures & Loop-back table (or "None") | (table rows) |
| `{{ARTIFACTS_CHECKED_ROWS}}` | Rows for the Artifacts Checked table | (table rows) |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Verification Report — {{APP_NAME}} v{{VERSION_NAME}}{{RERUN_LABEL}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Version | {{VERSION_NAME}} ({{VERSION_CODE}}) |
| Build commit | `{{BUILD_COMMIT}}` |
| Verifier | {{VERIFIER}} |
| Implementing session (must differ) | {{IMPLEMENTING_SESSION}} |
| Verification date | {{VERIFY_DATE}} |
| Factory version | {{FACTORY_VERSION}} |

<!-- The verifier and implementing session MUST be different (fresh-context verification). If they
     cannot be separated, record that as a waiver below — it does not silently pass. -->

## Overall Verdict

**{{OVERALL_VERDICT}}**

{{VERDICT_SUMMARY}}

<!-- Verdict is exactly one of PASS / PASS with waivers / FAIL (CONVENTIONS.md D3 — "CONDITIONAL"
     is abolished). PASS = every gate PASS. PASS with waivers = every gate PASS or covered by a
     recorded waiver below. FAIL = any gate FAIL with no waiver. Nothing ships on FAIL. The summary
     states which gates drove the verdict and, if PASS with waivers, names the waivers in one line.
     2–5 sentences. -->

## Gate Results

<!-- One row per gate in 07-checklists/release-readiness.md (plus the accessibility gate from
     07-checklists/accessibility.md). Result ∈ PASS / FAIL / N/A (N/A needs a justification in
     Evidence/notes). Command/method = the exact command or procedure run — a gate marked PASS
     without an executed check is not verified. Evidence/notes = artifact path, command exit, test
     count, or the reason for N/A. Every FAIL row must have a matching row in Failures & Loop-back.
     The rows below cover the mandatory gates; add store-specific rows as needed. -->

| Gate | Command / method | Result | Evidence / notes |
|---|---|---|---|
| Release bundle builds (clean) | `./gradlew clean :app:bundleRelease` exits 0; artifact present | PASS | `app/build/outputs/bundle/release/app-release.aab` | <!-- (example — replace) -->
| Build matrix (variants/ABIs build) | `./gradlew assembleRelease` across configured variants | PASS | 2 flavors × release built clean | <!-- (example — replace) -->
| Lint clean / baselined | `./gradlew :app:lintRelease` exits 0; count ≤ P4 baseline | PASS | 0 new, baseline 14 unchanged | <!-- (example — replace) -->
| Unit tests green (release variant) | `./gradlew :app:testReleaseUnitTest` exits 0 | PASS | 218 tests, 0 failures | <!-- (example — replace) -->
| Crash-free smoke (release build) | install R8 build; top-5 flows + monetization; `logcat -d *:E` clean | PASS | No FATAL/MissingClass on Pixel 8 API 36 | <!-- (example — replace) -->
| Upgrade-path install | install prior published APK, gen data, `adb install -r` over it, launch | PASS | Data intact, no crash (`02-engineering/11-data-migration-safety.md`) | <!-- (example — replace) -->
| Pseudo-locale sweep | `en-XA` + `ar-XB`; top flows, no truncation/RTL breakage | PASS | Screens captured; no clipping | <!-- (example — replace) -->
| Accessibility gate | `07-checklists/accessibility.md` full pass | PASS | TalkBack + contrast + 200% font OK | <!-- (example — replace) -->
| Startup time | macrobenchmark / `am start -W`; band per `08-knowledge/android/play-vitals-performance.md` | PASS | Cold 612 ms (Medium band) | <!-- (example — replace) -->
| Store-asset spec check | every STORE_LISTING Asset Manifest row = Y; dimensions vs store spec | PASS | googleplay/en-US assets all spec-compliant | <!-- (example — replace) -->
| Docs current | PROJECT_STATE P8/release tracker + CHANGELOG entry present and dated | PASS | Both updated {{VERIFY_DATE}} | <!-- (example — replace) -->
{{GATE_RESULT_ROWS}}

## Waivers

<!-- A waiver lets a non-PASS condition ship anyway, with explicit owner accountability. Each
     waiver: WHAT (the gate/finding waived), WHY ACCEPTABLE (the risk rationale — why shipping is
     safe), OWNER (the human who accepts the risk), EXPIRY (the version/date by which it must be
     resolved). Every waiver also lands in docs/factory/reports/RISK_REGISTER.md. A verdict of
     "PASS with waivers" requires at least one row here; "PASS" requires zero. Empty = "None". -->

| What (gate/finding) | Why acceptable | Owner | Expiry |
|---|---|---|---|
| Startup band Medium not Good | 612 ms within Play vitals Good-adjacent; tracked as ENG-021 | Owner (name) | v4.1.0 / 2026-08-01 | <!-- (example — replace) -->
{{WAIVER_ROWS}}

## Failures & Loop-back

<!-- One row per FAIL in Gate Results. Routing follows 01-core/PROJECT_LIFECYCLE.md P8 loop-back
     rules: code defect → P4; UI defect → P6; asset/listing defect → P7. Name the finding/ticket
     the FAIL maps to (ENG-/DES-/A11Y-/ASO-/IMPL-/COPY-xxx) and the phase it routes to. After the
     fix and a rebuilt candidate, verification re-runs as _r2/_r3 (a rebuilt artifact invalidates
     the Build and Quality gates entirely). Empty on a passing run — write "None". -->

| Failed gate | Finding / ticket | Routes to phase | Fix owner | Re-verify in |
|---|---|---|---|---|
| Crash-free smoke | IMPL-031 (R8 strips reflection class) | P4 (code defect) | Eng session | _r2 | <!-- (example — replace) -->
{{FAILURE_ROUTING_ROWS}}

## Artifacts Checked

<!-- The concrete artifacts this verification inspected, with their paths and the state confirmed.
     This is the audit trail proving the gates ran against the shipping build. Include the AAB and
     its checksum, mapping.txt archive, the STORE_LISTING instances, the RELEASE handover, and the
     PROJECT_STATE / CHANGELOG entries. State present/absent and the verified value. -->

| Artifact | Path | Confirmed |
|---|---|---|
| Release AAB | `app/build/outputs/bundle/release/app-release.aab` | Present; SHA-256 a1b2c3… | <!-- (example — replace) -->
| R8 mapping | `docs/factory/reports/release-v{{VERSION_NAME}}/mapping.txt` | Archived | <!-- (example — replace) -->
| Store listing | `docs/factory/reports/STORE_LISTING_googleplay_en-US.md` | Finalized, all assets Y | <!-- (example — replace) -->
| Release handover | `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` | Validated PASS | <!-- (example — replace) -->
{{ARTIFACTS_CHECKED_ROWS}}

## Sign-off

| Field | Value |
|---|---|
| Verdict | {{OVERALL_VERDICT}} |
| Run by | {{VERIFIER}} |
| Date | {{VERIFY_DATE}} |
| Human signing confirmation (release key) | <!-- owner name + date, per release-readiness Build group --> |

On PASS or PASS with waivers: proceed to `01-core/prompts/16-release-preparation.md`; log this
report in `docs/factory/CHANGELOG.md`. On FAIL: route each failure per the table above, fix,
rebuild the candidate if code changed, and re-run verification as the next `_r<N>` instance.
