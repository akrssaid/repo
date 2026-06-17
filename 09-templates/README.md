# 09-templates — Template Library

This directory holds the ten document templates that turn factory work into durable per-project
artifacts. Templates are the only files in the factory allowed to contain `{{SNAKE_CASE}}`
placeholder tokens (the template mechanics are owned by `01-core/CONVENTIONS.md`). A prompt in
`01-core/prompts/` instantiates a template by copying it into the TARGET app repo under
`docs/factory/`, filling every token, and stripping the template scaffolding. The factory copy is
never edited during project work — it changes only via a factory release recorded in
`10-releases/FRAMEWORK_CHANGELOG.md`.

## Anatomy of a Template

Every template in this directory contains, in order:

1. **Title + purpose paragraph** — what the instantiated document is for.
2. **Usage section** — who instantiates it, during which prompt and lifecycle phase, and the exact
   instance path under `docs/factory/`.
3. **Field Guide table** — every `{{TOKEN}}` in the file, its meaning, and a realistic example value.
4. **Template body** — the full instantiable document below a `TEMPLATE BODY` marker comment,
   containing token-bearing header blocks, tables with exactly one literal example row marked
   `(example — replace)`, and `<!-- guidance comments -->` describing what good content looks like.

## Instantiation Doctrine (mandatory, every time)

The template mechanics (token convention, copy/fill/strip rules, the zero-token verification gate)
are owned by `01-core/CONVENTIONS.md`; the steps below are the operational checklist that applies
each time a template is instantiated.

1. **Copy** everything below the template's `TEMPLATE BODY` marker comment into the instance path
   listed in the template's Usage section (create parent directories if missing).
2. **Fill every token.** Replace each `{{TOKEN}}` with a real value per the Field Guide. Never
   leave a token because the value is unknown — write `None`, `N/A`, or an explicit finding instead.
3. **Delete every guidance comment.** No `<!-- ... -->` blocks may survive into the instance.
4. **Delete every example row.** Rows marked `(example — replace)` exist to show shape and tone;
   replace them with real rows, never ship them.
5. **Verify zero tokens remain** before declaring the artifact done. Both commands grep for the
   literal `{{` token marker and must return ZERO hits:

   ```bash
   grep -rn '\{\{' docs/factory/ && echo "FAIL: unfilled tokens" || echo "PASS"
   ```

   ```powershell
   if (Get-ChildItem docs/factory -Recurse -Filter *.md | Select-String -Pattern '\{\{') `
     { 'FAIL: unfilled tokens' } else { 'PASS' }
   # PASS = zero hits. Any hit = unfilled token, fix before proceeding.
   ```

6. **Record the instantiation** in the target repo's `docs/factory/CHANGELOG.md` per the
   instantiating prompt's Documentation Update Rules.

An instance that still contains `{{`, a guidance comment, or an example row fails
`07-checklists/handover-validation.md` and must not be consumed by downstream prompts.

## Token Convention

- Tokens are `{{UPPER_SNAKE_CASE}}` — double braces, no spaces inside braces.
- Tokens hold **values** (names, dates, counts, verdicts), never whole sections. Section prose is
  written fresh by the instantiating agent following the guidance comments.
- Multi-line free-form areas use a single token (e.g. `{{FLOW_MAP_ASCII}}`) whose Field Guide entry
  says what to put there.
- Every token in a template body MUST appear in that template's Field Guide table, and vice versa.

## Template Index

| Template | Instantiated by | Phase | Instance path under target repo |
|---|---|---|---|
| `engineering-audit-report.md` | `01-core/prompts/03-modernization-audit.md` | P3 | `docs/factory/audits/ENGINEERING_AUDIT.md` |
| `design-audit-report.md` | `01-core/prompts/05-design-audit.md` | P5 | `docs/factory/audits/DESIGN_AUDIT.md` |
| `screen-inventory.md` | `01-core/prompts/05-design-audit.md` via `03-design/02-screen-inventory.md` | P5 | `docs/factory/audits/SCREEN_INVENTORY.md` |
| `aso-audit-report.md` | `01-core/prompts/12-aso-audit.md` | P7 | `docs/factory/audits/ASO_AUDIT.md` |
| `keyword-map.md` | `01-core/prompts/13-keyword-research.md` | P7 | `docs/factory/reports/KEYWORD_MAP.md` |
| `store-listing.md` | `01-core/prompts/14-store-assets.md` | P7 | `docs/factory/reports/STORE_LISTING_<store>_<locale>.md` (one per store × locale) |
| `verification-report.md` | `01-core/prompts/15-final-verification.md` | P8 | `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (re-runs append `_r2`, `_r3`) |
| `release-report.md` | `01-core/prompts/16-release-preparation.md` | P8 | `docs/factory/reports/RELEASE_REPORT_v<versionName>.md` |
| `risk-register.md` | `01-core/prompts/03-modernization-audit.md` (created P3); updated at every phase gate | P3–P8 | `docs/factory/reports/RISK_REGISTER.md` |
| `experiment-log.md` | `01-core/prompts/14-store-assets.md` (seeded); driven by `04-aso/workflows/experiment-program.md` (also icon/screenshot workflows) | P7–post-release | `docs/factory/reports/EXPERIMENT_LOG.md` |

## Rules of the Road

- **One instance per template per project**, except `store-listing.md` (one per store × locale),
  `release-report.md` and `verification-report.md` (one per release; verification re-runs append
  `_r2`, `_r3`).
- **Instances are living documents** where their template says so (`risk-register.md`,
  `screen-inventory.md` status columns); otherwise they are snapshots — re-running a prompt
  produces a new dated instance or an explicit revision noted in the header block.
- **Do not invent new tokens** when instantiating. If a template lacks a field you need, add the
  content as plain prose in the instance and file a factory improvement note in
  `10-releases/FRAMEWORK_CHANGELOG.md` candidates.
- Scales used in all templates are the canonical ones: Severity = Critical/High/Medium/Low;
  Effort = S/M/L/XL; Status = Not Started/In Progress/Blocked/Done/Deferred.
