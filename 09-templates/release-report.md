# Template — Release Report

This template produces the permanent record of one production release: what shipped, what was
verified, the exact artifacts and their checksums, the staged-rollout decision log, and the
retrospective. The instance is both the audit trail (which build is live, where the mapping file
is) and the input to the next cycle's planning — Retro Notes feed factory improvements and the
next roadmap.

## Usage

- **Instantiated by:** the release manager agent running
  `01-core/prompts/16-release-preparation.md`, after `15-final-verification.md` has produced a
  passing verification report; rollout and post-release sections are updated live during the
  rollout per `10-releases/release-workflow.md`.
- **Phase:** P8 — Verification & Release.
- **Instance path:** `docs/factory/reports/RELEASE_REPORT_v{{VERSION_NAME}}.md` in the TARGET app
  repo — one instance per release (e.g. `RELEASE_REPORT_v4.0.0.md`).
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain. Rollout Log and
  post-release sections may legitimately grow after initial instantiation.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / applicationId | Solar Camera Pro / com.solarcamera.pro |
| `{{VERSION_NAME}}` / `{{VERSION_CODE}}` | versionName / versionCode shipped (per `10-releases/app-versioning-policy.md`) | 4.0.0 / 40000 |
| `{{GIT_TAG}}` / `{{COMMIT_SHA}}` | Release tag and the SHA it points to | v4.0.0 / 9f3c1a7e… |
| `{{RELEASE_DATE}}` | Date rollout started, ISO 8601 | 2026-06-15 |
| `{{RELEASE_MANAGER}}` | Who ran the release (agent role + owner) | Release agent, approved by owner |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{SCOPE_SUMMARY}}` | One-paragraph summary of the release's purpose | Modernization release: SDK 36, M3 redesign |
| `{{AAB_PATH}}` / `{{AAB_SHA256}}` | Path to the shipped AAB / its SHA-256 | `app/release/app-release.aab` / a1b2… |
| `{{AAB_SIZE_MB}}` / `{{SIZE_DELTA}}` | Artifact size / delta vs previous release | 28.4 MB / +1.1 MB (+4%) |
| `{{MAPPING_ARCHIVED}}` | Y/N — R8 mapping.txt archived with location | Y — `docs/factory/reports/mappings/v4.0.0/` |
| `{{PREV_VERSION_NAME}}` | Previous live version for comparison | 3.6.2 |
| `{{STORE_STATUS_*}}` | Per-store submission status | Live (100%) |
| `{{RETRO_*}}` | Retrospective bullets (went well / improve / factory changes) | (bullets) |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Release Report — {{APP_NAME}} v{{VERSION_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Version | {{VERSION_NAME}} ({{VERSION_CODE}}) |
| Git tag | `{{GIT_TAG}}` → `{{COMMIT_SHA}}` |
| Release date | {{RELEASE_DATE}} |
| Release manager | {{RELEASE_MANAGER}} |
| Factory version | {{FACTORY_VERSION}} |
| Previous live version | {{PREV_VERSION_NAME}} |

## Release Scope

{{SCOPE_SUMMARY}}

<!-- Then enumerate what shipped, grouped as below. Every line references its
     docs/factory/CHANGELOG.md entry — nothing ships that is not in the changelog. Migrations
     = toolchain/SDK/data migrations completed this cycle (link the 02-engineering/migrations/
     guide used). -->

**Features:**
- <!-- one line per feature + CHANGELOG ref -->

**Fixes:**
- <!-- one line per user-visible fix + CHANGELOG ref -->

**Migrations:**
- <!-- e.g. "targetSdk 33 → 36 (02-engineering/migrations/android-sdk-migration.md) — CHANGELOG 2026-05-28" -->

## Verification Summary

Full report: `docs/factory/reports/VERIFICATION_REPORT_v{{VERSION_NAME}}.md` (from prompt 15).

<!-- Mirror the gate results only — detail lives in the verification report. Every gate from
     07-checklists/release-readiness.md appears here. A FAIL row with a shipped release requires
     an explicit owner-approved waiver note in the Notes column and a RISK entry. -->

| Gate | Result | Notes |
|---|---|---|
| Release build assembles (`./gradlew :app:bundleRelease`) | PASS | — | <!-- (example — replace) -->

## Artifacts

| Artifact | Value |
|---|---|
| AAB path | `{{AAB_PATH}}` |
| AAB SHA-256 | `{{AAB_SHA256}}` |
| Size | {{AAB_SIZE_MB}} ({{SIZE_DELTA}} vs v{{PREV_VERSION_NAME}}) |
| mapping.txt archived | {{MAPPING_ARCHIVED}} |
| versionCode | {{VERSION_CODE}} |

<!-- Checksum command (PowerShell): Get-FileHash -Algorithm SHA256 app\release\app-release.aab
     mapping.txt MUST be archived before rollout — without it, post-release crash stacks are
     unreadable. A size delta > +10% needs one sentence of explanation here. -->

## Rollout Log

<!-- One row per staged-rollout decision point, per 10-releases/release-workflow.md stages.
     Vitals = crash rate / ANR rate at decision time from Play Console (record actual numbers,
     not "fine"). Decision ∈ Advance / Hold / Rollback — Hold and Rollback rows must say why in
     a following bullet. Decided by = owner or agent-with-owner-approval. -->

| Stage | Date | Crash rate | ANR rate | Decision | Decided by |
|---|---|---|---|---|---|
| 10% | 2026-06-15 | 0.21% | 0.04% | Advance | Owner | <!-- (example — replace) -->

## Store Status

<!-- One row per store this release targets. Status: Submitted / In Review / Approved /
     Live (n%) / Rejected. Listing column links the STORE_LISTING instance submitted with it. -->

| Store | Status | Listing instance |
|---|---|---|
| Google Play | {{STORE_STATUS_GOOGLE_PLAY}} | `docs/factory/reports/STORE_LISTING_googleplay_en-US.md` |

## Issues Found Post-Release

<!-- Anything discovered after rollout started: crash clusters, review complaints, store policy
     flags. Severity uses the canonical scale. Resolution: Fixed in vX / Hotfix shipped /
     Accepted (→ RISK_REGISTER.md ref) / Monitoring. "None as of <date>" is valid and must be
     dated, not assumed. -->

| ID | Found | Severity | Description | Resolution |
|---|---|---|---|---|
| REL-001 | 2026-06-16 | High | Crash cluster `NullPointerException` in editor on API 26 devices (0.9% of sessions) | Hotfix v4.0.1, rolled out 2026-06-18 | <!-- (example — replace) -->

## Retro Notes

<!-- Written when rollout reaches 100% (or after rollback). Three lists, 2–5 bullets each,
     specific enough to act on next cycle. Factory-level improvements also get filed as
     candidates in 10-releases/FRAMEWORK_CHANGELOG.md. -->

**What went well:**
- {{RETRO_WELL}}

**What to improve next cycle:**
- {{RETRO_IMPROVE}}

**Factory changes to propose:**
- {{RETRO_FACTORY}}
