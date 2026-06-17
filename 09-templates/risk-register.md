# Template — Risk Register

This template produces the project's single living risk ledger: every threat to the modernization
effort — technical, policy, schedule, monetization — with likelihood, impact, mitigation, and an
early-warning trigger. Unlike the audit templates, the instance is created once at P2 and updated
at every phase gate until release; it is the document that stops known dangers from being
rediscovered the hard way during a migration or a store review.

## Usage

- **Instantiated by:** the project-memory agent running `01-core/prompts/02-project-memory.md`,
  seeded from Discovery findings (`docs/factory/audits/DISCOVERY.md`).
- **Phase:** created at P2; **updated at every phase gate P3–P8** and whenever a migration or
  store submission begins (see Review Cadence section inside the instance).
- **Instance path:** `docs/factory/reports/RISK_REGISTER.md` in the TARGET app repo — exactly one
  per project, never versioned-per-phase; history lives in the Retired Risks log.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and BOTH example rows, verify zero `{{` remain. At P2 the Active Risks
  table may legitimately be short (3–6 seed risks); it must never be empty.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / applicationId | Solar Camera Pro / com.solarcamera.pro |
| `{{CREATED_DATE}}` | Date register created at P2, ISO 8601 | 2026-06-10 |
| `{{LAST_REVIEW_DATE}}` | Date of most recent review (update on every edit) | 2026-06-10 |
| `{{LAST_REVIEW_PHASE}}` | Phase gate of the most recent review | P3 gate |
| `{{OWNER_DEFAULT}}` | Default risk owner when not otherwise assigned | Owner (solo publisher) |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{MATRIX_CELL_*}}` | RISK-xxx IDs placed in each of the 9 likelihood×impact matrix cells ("—" if empty) | RISK-001, RISK-004 |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Risk Register — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Created | {{CREATED_DATE}} (P2) |
| Last reviewed | {{LAST_REVIEW_DATE}} at {{LAST_REVIEW_PHASE}} |
| Default owner | {{OWNER_DEFAULT}} |
| Factory version | {{FACTORY_VERSION}} |

## Active Risks

<!-- One row per open risk. Rules:
     - ID: RISK-001 ascending, permanent, never reused.
     - Category ∈ technical / policy / schedule / monetization.
     - Likelihood and Impact ∈ H/M/L; Exposure = the matrix cell, written "L×I" (e.g. "M×H").
       H×H and H×M risks must have mitigation IN PROGRESS, not just planned.
     - Description states the risk as an event: "X happens, causing Y" — not a vague worry.
     - Trigger = the observable early-warning sign that says the risk is materializing; if you
       cannot name a trigger you do not understand the risk yet.
     - Status ∈ Open / Mitigating / Watching.
     - Every Deferred audit finding and every accepted-risk waiver elsewhere in docs/factory/
       MUST have a row here pointing back to its source artifact. -->

| ID | Category | Description | Likelihood | Impact | Exposure | Mitigation | Trigger / early warning | Owner | Status |
|---|---|---|---|---|---|---|---|---|---|
| RISK-001 | technical | R8 full-mode in AGP 8.x strips reflection-accessed classes used by the legacy PDF SDK, causing release-only crashes | M | H | M×H | Add keep rules per `02-engineering/migrations/agp-migration.md`; verify release build smoke test on every migration step | Release-build smoke test crashes where debug passes; `ClassNotFoundException` in release stacktrace | Owner | Mitigating | <!-- (example — replace) -->
| RISK-002 | policy | Play review rejects the update for undeclared `MANAGE_EXTERNAL_STORAGE` permission, blocking the release train | M | H | M×H | Replace with scoped storage / SAF before submission per `08-knowledge/stores/policy-landmines.md`; pre-submit permission declaration review | Permission still present in merged manifest at P8 gate; Play Console pre-launch warning | Owner | Open | <!-- (example — replace) -->

## Risk Matrix

<!-- Place every Active RISK-xxx ID in its cell; "—" for empty cells. Keep the grid current with
     the table — a reviewer reads this first. Anything in the top-right (H likelihood × H impact)
     means: stop feature work, mitigate now. -->

```text
                         IMPACT
              Low          Medium         High
         ┌─────────────┬─────────────┬─────────────┐
  High   │ {{MATRIX_CELL_HL}} │ {{MATRIX_CELL_HM}} │ {{MATRIX_CELL_HH}} │
L        ├─────────────┼─────────────┼─────────────┤
I Medium │ {{MATRIX_CELL_ML}} │ {{MATRIX_CELL_MM}} │ {{MATRIX_CELL_MH}} │
K        ├─────────────┼─────────────┼─────────────┤
E Low    │ {{MATRIX_CELL_LL}} │ {{MATRIX_CELL_LM}} │ {{MATRIX_CELL_LH}} │
         └─────────────┴─────────────┴─────────────┘
```

### Exposure Response Policy

<!-- Standing policy — keep verbatim in the instance. It converts a matrix position into a
     required behavior, so triage is never re-debated per risk. -->

| Exposure band | Cells | Required response |
|---|---|---|
| Severe | H×H | Stop other work; mitigate immediately; phase gates blocked until downgraded |
| Elevated | H×M, M×H | Mitigation must be In Progress with a named next action and date |
| Moderate | H×L, M×M, L×H | Mitigation planned; re-score at every phase gate |
| Low | M×L, L×M, L×L | Watch only; trigger monitoring is sufficient |

## Retired Risks

<!-- A risk leaves the Active table only by moving here with an outcome:
     - materialized — it happened; record what it cost and link the incident/finding/report;
     - avoided — mitigation worked or the cause was removed (say which);
     - expired — the window passed (e.g. the migration completed without it occurring).
     Never delete rows from this log; it is the project's institutional memory. -->

| ID | Description (short) | Outcome | Date | Notes |
|---|---|---|---|---|
| RISK-000 | Gradle 8 upgrade breaks legacy `buildSrc` plugins | avoided | 2026-05-20 | Plugins migrated to version catalog first; upgrade clean — see `ENGINEERING_AUDIT.md` ENG-002 | <!-- (example — replace) -->

## Review Cadence

<!-- Keep these rules verbatim in the instance — they are operating procedure, not commentary. -->

1. **Every phase gate (P2→P8):** review every Active row — re-score likelihood/impact, update the
   matrix, retire what no longer applies, and update the header's Last reviewed fields. A phase
   gate does not pass with an unreviewed register (enforced by
   `07-checklists/release-readiness.md` at P8 and by `01-core/PROJECT_LIFECYCLE.md` gates).
2. **Before any migration starts** (`02-engineering/migrations/*`): add that migration's known
   risks from the guide's risk section BEFORE the first commit of the migration.
3. **Before any store submission:** add submission-specific risks (policy, rejection, rollout)
   from `08-knowledge/stores/policy-landmines.md` BEFORE submitting.
4. **When a trigger fires:** set Status to Mitigating the same day and note the observation in
   `docs/factory/PROJECT_STATE.md`; if the risk materializes, retire it as materialized and file
   the resulting work as findings/tasks.
5. **Method reference:** scoring and mitigation doctrine follow the Risk Management section of
   `BEST_PRACTICES.md`.
6. Log every register update in `docs/factory/CHANGELOG.md` (date, rows touched, reason).
