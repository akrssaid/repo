# Template — ASO Audit Report

This template produces the App Store Optimization baseline for a target app: the current listing
captured verbatim per store, a scored assessment, evidenced findings, competitor intelligence, and
keyword gaps. The instance is the factual foundation for
`01-core/prompts/13-keyword-research.md` (keyword map) and `01-core/prompts/14-store-assets.md`
(new listings) — capture-before-judgment is the rule: record what exists exactly, then score it.

## Usage

- **Instantiated by:** the ASO analyst agent running `01-core/prompts/12-aso-audit.md` (with store
  specifics from `04-aso/stores/*.md` and methods from `04-aso/workflows/competitor-analysis.md`).
- **Phase:** P7 — ASO & Store Assets.
- **Instance path:** `docs/factory/audits/ASO_AUDIT.md` in the TARGET app repo.
- **Scope rule:** repeat the per-store snapshot subsection once per store in scope (Google Play,
  Huawei AppGallery, RuStore); delete subsections for stores not in scope.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / store package id | Solar Camera Pro / com.solarcamera.pro |
| `{{AUDIT_DATE}}` | Date audit completed, ISO 8601 | 2026-06-10 |
| `{{AUDITOR_ROLE}}` | Persona from prompt 12 Role section | Senior ASO Strategist |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{STORES_IN_SCOPE}}` | Comma-separated stores audited | Google Play, RuStore |
| `{{LOCALES_IN_SCOPE}}` | Locales whose listings were captured | en-US, ru-RU |
| `{{ASO_VERDICT}}` | One-line verdict: Strong / Adequate / Weak / Neglected + qualifier | Neglected — listing untouched since 2023 |
| `{{STORE_NAME}}` | Store for a snapshot subsection | Google Play |
| `{{FIELD_VALUE_*}}` / `{{FIELD_CHARS_*}}` | Verbatim listing field text / its character count | Solar Camera — Night Photos / 27 |
| `{{RATING_AVG}}` / `{{RATING_COUNT}}` | Current average rating / total ratings on that store | 4.1 / 12,408 |
| `{{RANK_BASELINE_DATE}}` | Date the head-term rank estimates were taken, ISO 8601 | 2026-06-10 |
| `{{RANK_BASELINE_ROWS}}` | Keyword-rank baseline rows (head term, est. rank, method) | (table rows) |
| `{{REVIEW_MINING_ROWS}}` | Own-app review-mining rows (theme, frequency, sentiment, excerpt) | (table rows) |
| `{{SCORE_*}}` | 1–5 score per scorecard dimension | 2 |
| `{{KEYWORD_GAP_SUMMARY}}` | Prose summary of unexploited keyword territory | (prose) |
| `{{KEYWORD_RESEARCH_HINTS}}` / `{{STORE_ASSET_HINTS}}` | Directive handoff lists for prompts 13 / 14 | (bullet list) |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# ASO Audit — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Audit date | {{AUDIT_DATE}} |
| Auditor role | {{AUDITOR_ROLE}} |
| Factory version | {{FACTORY_VERSION}} |
| Stores in scope | {{STORES_IN_SCOPE}} |
| Locales in scope | {{LOCALES_IN_SCOPE}} |

## Executive Summary

**ASO verdict:** {{ASO_VERDICT}}

<!-- 3–6 sentences. State how discoverable and how convertible the listing is today, the single
     biggest missed opportunity, and any policy exposure (per
     08-knowledge/stores/policy-landmines.md). Quantify: char-budget utilization, rating trend,
     screenshot age. -->

## Current Listing Snapshot — {{STORE_NAME}}

<!-- CAPTURE VERBATIM. Copy the live listing text exactly, typos included — this is evidence,
     not draft copy. Repeat this whole H2 once per store in scope. Char counts are actual/limit
     per the store's spec in 04-aso/stores/. For locales beyond the primary, add an H3 per
     locale with the same field list. -->

### Metadata (as live on {{AUDIT_DATE}}, locale {{PRIMARY_LOCALE}})

| Field | Current value (verbatim) | Chars used / limit |
|---|---|---|
| Title | {{FIELD_VALUE_TITLE}} | {{FIELD_CHARS_TITLE}}/30 |
| Short description | {{FIELD_VALUE_SHORT}} | {{FIELD_CHARS_SHORT}}/80 |
| Full description | see fenced block below | {{FIELD_CHARS_FULL}}/4000 |
| Developer name | {{FIELD_VALUE_DEVELOPER}} | — |
| Category | {{FIELD_VALUE_CATEGORY}} | — |

```text
{{FIELD_VALUE_FULL_DESCRIPTION}}
```

### Ratings & Reviews

- Average rating: {{RATING_AVG}} from {{RATING_COUNT}} ratings; trend over last 90 days: {{RATING_TREND}}

**Own-app review mining.** <!-- Mine the app's OWN recent reviews and rank themes by frequency,
the same method used for competitor analysis (04-aso/workflows/competitor-analysis.md) and the
same raw material as DESIGN_AUDIT's User Insight section — keep them consistent. One row per
recurring theme, ordered by frequency. Frequency = approximate share of reviews in the sampled
window ("~30% of 1–2★"). Sentiment ∈ Complaint / Praise / Mixed. Excerpt = one verbatim quote
(typos kept) + its star rating. Themes drive both copy/keyword choices and ASO findings below. -->

| Theme | Frequency | Sentiment | Representative excerpt (verbatim + ★) |
|---|---|---|---|
| Crashes on launch after update | ~28% of 1–2★ | Complaint | "keeps force closing since the last update, useless now" (1★) | <!-- (example — replace) -->
{{REVIEW_MINING_ROWS}}

### Keyword Rank Baseline (as of {{RANK_BASELINE_DATE}})

<!-- Estimated current ranking for the head terms this app should own, captured BEFORE any ASO
     change so post-launch movement is measurable. Ranks are ESTIMATES (agents cannot query live
     rank trackers) — derive from store-search position observed manually and flag accordingly;
     the same estimate-disclosure rule as KEYWORD_MAP.md applies. One row per head term. This block
     is the rank baseline that ASO_HANDOVER's Baseline Metrics Snapshot and the experiment log read.
     "Not ranking" is a valid and important value. -->

| Head term | Store | Est. rank | Method / confidence |
|---|---|---|---|
| night camera | Google Play | ~#40 (est.) | Manual search position, en-US, low confidence | <!-- (example — replace) -->
{{RANK_BASELINE_ROWS}}

### Asset Inventory

<!-- Every live visual asset: icon, feature graphic, each screenshot, video. Dimensions as
     actually served. Note age if determinable (old UI visible = stale). -->

| Asset | Count | Dimensions | Notes |
|---|---|---|---|
| Phone screenshots | 4 | 1080×1920 | Show pre-2024 UI; no captions; portrait only | <!-- (example — replace) -->

## Scorecard

<!-- Score 1 (absent/harmful) to 5 (best-in-class) across stores; if stores diverge sharply,
     note it in the column. Notes must name the evidence, not restate the score. -->

| Dimension | Score (1–5) | Notes |
|---|---|---|
| Metadata keyword coverage | {{SCORE_KEYWORDS}} | {{SCORE_KEYWORDS_NOTES}} |
| Conversion assets (icon/screens/video) | {{SCORE_ASSETS}} | {{SCORE_ASSETS_NOTES}} |
| Ratings health | {{SCORE_RATINGS}} | {{SCORE_RATINGS_NOTES}} |
| Localization depth | {{SCORE_LOCALIZATION}} | {{SCORE_LOCALIZATION_NOTES}} |
| Policy risk | {{SCORE_POLICY}} | {{SCORE_POLICY_NOTES}} |

## Findings Register

<!-- One row per finding, ID ASO-001 ascending (zero-padded 3 per CONVENTIONS.md D2), ordered by
     severity. Store column names which
     store(s) the finding applies to ("All" allowed). Evidence = the snapshot section/row above
     or a screenshot path under docs/factory/assets/. Severity = Critical/High/Medium/Low.
     Policy-risk findings are always Critical or High. -->

| ID | Severity | Store | Evidence | Recommendation | Status |
|---|---|---|---|---|---|
| ASO-001 | High | Google Play | Title snapshot — 27/30 chars, zero category keywords | Rebuild title around primary keyword cluster from `KEYWORD_MAP.md` | Not Started | <!-- (example — replace) -->

## Competitor Summary

<!-- 4–6 direct competitors from the same category/keyword space, per
     04-aso/workflows/competitor-analysis.md. "Screenshot hook" = the message of their first
     screenshot — that is the conversion battleground. Capture strategy, not just facts. -->

| Competitor | Title strategy | Icon approach | Screenshot hook | Rating |
|---|---|---|---|---|
| NightCam Pro | Brand + 2 keywords ("night camera") | Dark bg, single lens glyph, high contrast | Before/after low-light shot, caption overlay | 4.5 (89k) | <!-- (example — replace) -->

## Keyword Gap Summary

{{KEYWORD_GAP_SUMMARY}}

<!-- Prose + a short list: keyword territories competitors rank for that our listing does not
     even mention; branded terms we own but underuse; locale gaps. This is a hypothesis list —
     prompt 13 will validate and size it. Do NOT publish volume numbers here; flag every
     estimate as an estimate. -->

## Inputs for Keyword Research & Store Assets (Prompts 13/14 handoff)

- **Keyword research starting points (prompt 13):** {{KEYWORD_RESEARCH_HINTS}}
  <!-- Seed terms, competitor terms to mine, locales to prioritize. -->
- **Store asset priorities (prompt 14):** {{STORE_ASSET_HINTS}}
  <!-- Ordered: e.g. 1. Replace stale screenshots post-redesign (pair with prompt 11 output),
       2. Feature graphic, 3. Icon A/B candidate from prompt 10. -->
- **Policy items to clear before any listing update:** <!-- list ASO-xxx policy findings; these
  block submission and must also appear in docs/factory/reports/RISK_REGISTER.md. -->

> **Experiment logging note (do NOT embed experiments here):** any listing test proposed off this
> audit is logged and tracked in `docs/factory/reports/EXPERIMENT_LOG.md`
> (`09-templates/experiment-log.md`), driven by `04-aso/workflows/experiment-program.md`. This
> audit only seeds hypotheses; it never holds the experiment table.
