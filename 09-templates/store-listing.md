# Template — Store Listing

This template produces the final, submission-ready listing for one store × locale pair: every
metadata field as final copy with verified character counts, the asset manifest, the What's New
text, and the compliance attestations that make the listing safe to submit. The instance is what
actually gets pasted into the store console — copy here is final, not draft (drafts live in
`KEYWORD_MAP.md`).

## Usage

- **Instantiated by:** the store-assets agent running `01-core/prompts/14-store-assets.md`
  (field rules and limits from `04-aso/stores/<store>.md`; copy built from
  `docs/factory/reports/KEYWORD_MAP.md`).
- **Phase:** P7 — ASO & Store Assets; Publish Status section is completed during P8 release.
- **Instance path:** `docs/factory/reports/STORE_LISTING_<store>_<locale>.md` in the TARGET app
  repo — **one instance per store × locale** (e.g. `STORE_LISTING_googleplay_en-US.md`).
- **Store variance:** the field subsections below follow Google Play's field set. For Huawei
  AppGallery or RuStore, rename/add/remove field subsections to match that store's field list in
  `04-aso/stores/huawei-appgallery.md` / `04-aso/stores/rustore.md`, keeping the same
  copy-block + char-count + keywords-covered structure per field.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / package id | Solar Camera Pro / com.solarcamera.pro |
| `{{STORE}}` / `{{LOCALE}}` | Store and locale of this instance | Google Play / en-US |
| `{{LISTING_VERSION}}` | Listing revision number, v1 ascending per store×locale | v2 |
| `{{APP_VERSION_NAME}}` | App version this listing accompanies | 4.0.0 |
| `{{AUTHOR_ROLE}}` | Persona from prompt 14 Role section | Senior ASO Copywriter |
| `{{LISTING_DATE}}` | Date copy finalized, ISO 8601 | 2026-06-10 |
| `{{TITLE_FINAL}}` etc. | Final copy per field (fenced blocks) | Solar Camera: Night Camera |
| `{{TITLE_CHAR_COUNT}}` etc. | Actual character count of the final copy for that field | 26 |
| `{{TITLE_KEYWORDS}}` etc. | Keywords from `KEYWORD_MAP.md` covered by that field | night camera |
| `{{WHATS_NEW_FINAL}}` / `{{WHATS_NEW_CHAR_COUNT}}` | Release-notes copy / its char count | (copy) / 312 |
| `{{CATEGORY_PRIMARY}}` / `{{CATEGORY_SECONDARY}}` | Chosen primary / secondary store category ("—" if none) | Photography / — |
| `{{TAGS_CHOSEN}}` | Store tags selected (Play: up to 5), comma-separated | Camera, Photo Editor, Night Mode |
| `{{CATEGORY_TAGS_RATIONALE}}` | Why this category/tag set, tied to KEYWORD_MAP clusters | (prose) |
| `{{AB_VARIANT_ROWS}}` | A/B variant rows queued for the experiment log (EXP-xxx) | (table rows) |
| `{{POLICY_SWEEP_DATE}}` | Date listing was checked against `08-knowledge/stores/policy-landmines.md` | 2026-06-10 |
| `{{TRADEMARK_CHECK_RESULT}}` | Outcome of trademark scan over all copy | PASS — no third-party marks |
| `{{SUBMIT_DATE}}` / `{{APPROVE_DATE}}` / `{{LIVE_DATE}}` | Console milestone dates, ISO 8601 ("pending" until known) | 2026-06-12 |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Store Listing — {{APP_NAME}} — {{STORE}} ({{LOCALE}})

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Store / locale | {{STORE}} / {{LOCALE}} |
| Listing version | {{LISTING_VERSION}} |
| For app version | {{APP_VERSION_NAME}} |
| Author role | {{AUTHOR_ROLE}} |
| Finalized | {{LISTING_DATE}} |
| Sources | `docs/factory/reports/KEYWORD_MAP.md`, `docs/factory/audits/ASO_AUDIT.md` |

## Listing Fields

<!-- Per field: final copy in a fenced block (exactly what gets pasted into the console — no
     markdown formatting inside unless the store renders it), then the char-count line, then
     keywords covered. The char count must be counted, not eyeballed; limits per
     04-aso/stores/<store>.md. A field over budget is a defect, not a rounding error.
     Writing bar: first 250 chars of the full description carry the conversion message
     (08-knowledge/aso/conversion-optimization.md); benefits before features; no keyword
     stuffing; every claim must be true of the shipped app version. -->

### Title

```text
{{TITLE_FINAL}}
```

Chars: {{TITLE_CHAR_COUNT}}/30 — Keywords covered: {{TITLE_KEYWORDS}}

### Short Description

```text
{{SHORT_DESCRIPTION_FINAL}}
```

Chars: {{SHORT_DESCRIPTION_CHAR_COUNT}}/80 — Keywords covered: {{SHORT_DESCRIPTION_KEYWORDS}}

### Full Description

```text
{{FULL_DESCRIPTION_FINAL}}
```

Chars: {{FULL_DESCRIPTION_CHAR_COUNT}}/4000 — Keywords covered: {{FULL_DESCRIPTION_KEYWORDS}}

## Category & Tags

<!-- The category and tag choices are ASO levers, not afterthoughts (CONVENTIONS.md D16). Category
     options and tag limits come from 04-aso/stores/<store>.md (Play allows one primary + tags;
     verify there). Rationale must tie the choice to KEYWORD_MAP.md clusters and competitor
     placement from ASO_AUDIT.md — never pick a category by gut. -->

| Field | Value |
|---|---|
| Primary category | {{CATEGORY_PRIMARY}} |
| Secondary category | {{CATEGORY_SECONDARY}} |
| Tags | {{TAGS_CHOSEN}} |

**Rationale:** {{CATEGORY_TAGS_RATIONALE}}

## A/B Variants (EXP queue)

<!-- Variants for store experiments LIVE HERE (CONVENTIONS.md D16 — the ASO_HANDOVER copy is FINAL;
     alternatives belong in this file + the EXP queue, never in the handover). Each row is a
     queued listing experiment: an EXP-xxx ID (zero-padded 3 per D2), the surface it tests, the
     single variable changed, the control vs variant copy, and which KEYWORD_MAP cluster/hypothesis
     it probes. These rows are consumed by docs/factory/reports/EXPERIMENT_LOG.md
     (09-templates/experiment-log.md) — one live experiment per listing at a time (see
     04-aso/workflows/experiment-program.md); this table only queues them, it does not track
     results. Play allows up to 3 variants vs the current listing per experiment. "None planned"
     is valid. -->

| EXP-ID | Surface | Variable | Control (current) | Variant copy | Hypothesis / cluster |
|---|---|---|---|---|---|
| EXP-001 | Short description | Lead benefit vs lead keyword | {{SHORT_DESCRIPTION_FINAL}} | "Shoot stunning night photos — no tripod" | Benefit-led lead lifts CVR; KW-001 still present | <!-- (example — replace) -->
{{AB_VARIANT_ROWS}}

## Asset Manifest

<!-- One row per asset uploaded with this listing. Paths under
     docs/factory/assets/store/<store>/<locale>/ (canonical per CONVENTIONS.md D5 — the old
     assets/screenshots/store/, assets/feature-graphic/, assets/icons/ paths are abolished).
     Dimensions must match the store spec exactly (specs in 04-aso/stores/<store>.md); the
     Spec-compliant column is verified against actual file metadata, e.g.:
       PowerShell: Add-Type -AssemblyName System.Drawing;
                   [System.Drawing.Image]::FromFile("icon.png") | Select Width,Height
     Any N row blocks submission. Screenshots come from prompt 11 output and must show the
     CURRENT (post-redesign) UI. -->

| Asset type | Path | Dimensions | Spec-compliant |
|---|---|---|---|
| App icon | `docs/factory/assets/store/googleplay/en-US/icon-512.png` | 512×512 PNG, 32-bit | Y | <!-- (example — replace) -->

## What's New

```text
{{WHATS_NEW_FINAL}}
```

Chars: {{WHATS_NEW_CHAR_COUNT}}/500

<!-- Derive from docs/factory/CHANGELOG.md entries for {{APP_VERSION_NAME}}. Lead with the most
     user-visible change; plain language; no internal jargon or ticket IDs. -->

## Compliance Attestations

<!-- Every box must be checked (- [x]) before the listing may be submitted. An unchecked box is
     a hard stop. The signer is the agent/owner completing the sweep. -->

- [ ] **Claims-true check:** every capability claimed in title, descriptions, and screenshot
      captions exists and works in app version {{APP_VERSION_NAME}}.
- [ ] **Policy sweep** completed on {{POLICY_SWEEP_DATE}} against
      `08-knowledge/stores/policy-landmines.md` — no metadata violations (no ranking claims, no
      competitor references, no incentivized-review language, no restricted-content terms).
- [ ] **Trademark check:** {{TRADEMARK_CHECK_RESULT}} — no third-party brands in any copy field
      or asset text overlay (cross-check Excluded Keywords log in `KEYWORD_MAP.md`).
- [ ] **Asset spec check:** every Asset Manifest row is Y.
- [ ] **Screenshot truthfulness:** screenshots show real app UI from {{APP_VERSION_NAME}}; any
      device frames/captions do not obscure or fabricate functionality.
- [ ] **Localization check** (non-en locales): copy written/reviewed for {{LOCALE}}, not
      machine-translated verbatim (per `04-aso/workflows/localization.md`).

## Publish Status

<!-- Updated during P8 by prompt 16. Status ∈ Draft / Submitted / In Review / Approved / Live /
     Rejected. On rejection: record the store's stated reason verbatim, file a RISK or finding,
     and bump {{LISTING_VERSION}} for the corrected copy. -->

| Milestone | Status | Date |
|---|---|---|
| Submitted | {{SUBMIT_STATUS}} | {{SUBMIT_DATE}} |
| Approved | {{APPROVE_STATUS}} | {{APPROVE_DATE}} |
| Live | {{LIVE_STATUS}} | {{LIVE_DATE}} |

**Rejection log:** <!-- "None" or one bullet per rejection: date, store reason verbatim, fix, resubmit date. -->
