# Template — Keyword Map

This template produces the keyword strategy for a target app: the full keyword universe scored and
clustered, then mapped into concrete metadata slots per store with character budgets and draft
copy. The instance is the direct input to `01-core/prompts/14-store-assets.md` — listing copy is
written FROM the slot mapping, never invented at writing time. Honesty rule: agents cannot query
real search-volume data, so every volume/difficulty figure is an estimate and the document must say
so up front.

## Usage

- **Instantiated by:** the ASO analyst agent running `01-core/prompts/13-keyword-research.md`
  (method in `04-aso/workflows/keyword-research.md`; store field rules in `04-aso/stores/*.md`).
- **Phase:** P7 — ASO & Store Assets (after `ASO_AUDIT.md` exists).
- **Instance path:** `docs/factory/reports/KEYWORD_MAP.md` in the TARGET app repo.
- **Scope rule:** repeat the Slot Mapping subsection per store, and the Per-Locale section per
  non-primary locale in scope. Delete what does not apply.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / package id | Solar Camera Pro / com.solarcamera.pro |
| `{{MAP_DATE}}` | Date map completed, ISO 8601 | 2026-06-10 |
| `{{ANALYST_ROLE}}` | Persona from prompt 13 Role section | Senior ASO Strategist |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{STORES_IN_SCOPE}}` / `{{LOCALES_IN_SCOPE}}` | Stores / locales this map covers | Google Play / en-US, ru-RU |
| `{{METHOD_SOURCES}}` | Where keyword candidates came from | ASO_AUDIT competitor mining; Play autocomplete; review language |
| `{{STORE_NAME}}` / `{{LOCALE}}` | Store / locale for a mapping or locale section | Google Play / ru-RU |
| `{{PRIMARY_CLUSTER}}` | The KW-xxx cluster the title is built around | KW-001 night photography |
| `{{NEXT_REVIEW_DATE}}` | Date of next scheduled keyword refresh, ISO 8601 | 2026-09-10 |
| `{{REFRESH_TRIGGERS}}` | Event triggers that force an early refresh | Major competitor launch; rating drop below 4.0 |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Keyword Map — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Date | {{MAP_DATE}} |
| Analyst role | {{ANALYST_ROLE}} |
| Factory version | {{FACTORY_VERSION}} |
| Stores in scope | {{STORES_IN_SCOPE}} |
| Locales in scope | {{LOCALES_IN_SCOPE}} |
| Source audit | `docs/factory/audits/ASO_AUDIT.md` |

## Methodology Note

> **Estimate disclosure (mandatory, keep this block):** Volume and difficulty values in this
> document are qualitative estimates (H/M/L) derived from category knowledge, competitor listing
> analysis, and autocomplete signals — NOT from measured search data. They guide prioritization
> only. Where a real ASO tool measurement later becomes available, replace the estimate and note
> the source inline.

**Candidate sources used:** {{METHOD_SOURCES}}

<!-- List concretely how the universe was built: terms mined from competitor titles/descriptions
     in ASO_AUDIT.md, store autocomplete probes, vocabulary users employ in reviews, category
     browse terms, localization back-translations. -->

> **Canonical rules (cited, not restated):** seed and candidate counts follow
> `04-aso/workflows/keyword-research.md` (do not hardcode the numbers here — that file owns them);
> keyword density when placing terms in copy follows the density rule in
> `08-knowledge/aso/ranking-factors.md` (natural uses per primary keyword, hard cap). The Slot
> Mapping below must respect that density rule; cite it, never re-state the figures.

## Keyword Universe

<!-- Every candidate keyword that survived initial triage (seed/candidate target counts are owned
     by 04-aso/workflows/keyword-research.md — follow them there, do not hardcode here). Rules:
     - Relevance 1–5: 5 = the app IS this query; 1 = tangential (1–2s should instead go to the
       Excluded log — only keep them here if a cluster needs them as modifiers).
     - Volume/Difficulty: H/M/L estimates per the disclosure above.
     - Priority quadrant: WIN (high relevance + low/med difficulty), INVEST (high relevance +
       high difficulty), FILL (medium relevance, easy — use in long description), SKIP
       (log and exclude). Every WIN keyword must appear in at least one slot mapping below. -->

| Keyword | Cluster | Relevance (1–5) | Est. volume | Est. difficulty | Quadrant | Notes |
|---|---|---|---|---|---|---|
| night camera | KW-001 | 5 | H | H | INVEST | Head term; competitors NightCam Pro, Lumio own top slots | <!-- (example — replace) -->

## Cluster Summary

<!-- One row per cluster, each with a KW-xxx cluster ID (zero-padded 3 per CONVENTIONS.md D2 — the
     EXP queue and store-listing reference clusters by this ID). The PRIMARY cluster (exactly one)
     anchors the title; SECONDARY clusters get short-description and full-description sections; FILL
     clusters appear only in body copy. The Keyword Universe "Cluster" column uses these same IDs. -->

| Cluster ID | Theme | Role | Top keywords | Strategy |
|---|---|---|---|---|
| KW-001 | night photography | PRIMARY | night camera, low light camera | Anchor title + first description paragraph | <!-- (example — replace) -->

**Primary cluster:** {{PRIMARY_CLUSTER}}

## Slot Mapping — {{STORE_NAME}} ({{LOCALE}})

<!-- Repeat this H2 per store in scope. Char budgets come from 04-aso/stores/<store>.md — verify
     them there, do not trust memory. Draft copy here is the working draft; prompt 14 finalizes
     it in STORE_LISTING_<store>_<locale>.md. Char count must be the actual count of the draft.
     "Full description sections" = one row per planned paragraph/block, so keyword placement in
     the first 250 chars (above the fold) is deliberate. -->

| Field | Char budget | Assigned keywords | Draft copy | Chars |
|---|---|---|---|---|
| Title | 30 | night camera | Solar Camera: Night Camera | 26 | <!-- (example — replace) -->

## Per-Locale Notes — {{LOCALE}}

<!-- Repeat per non-primary locale. Never machine-translate the keyword list — keywords must be
     what native speakers actually type. Record: locale-specific terms with no English
     equivalent, terms that change cluster (e.g. a generic EN term is a brand in RU), and which
     slot mappings diverge from the primary locale and why. -->

{{LOCALE_NOTES}}

## Excluded Keywords

<!-- Every triaged-out keyword with the reason. Categories:
     trademark — competitor brand or third-party mark (NEVER place in metadata; Google Play
       trademark complaints suspend listings — see 08-knowledge/stores/policy-landmines.md);
     irrelevant — users searching it want something else;
     unwinnable — relevance fine, but entrenched head term we cannot rank for and cannot afford
       title space on. This log prevents re-litigating the same terms at the next refresh. -->

| Keyword | Reason | Detail |
|---|---|---|
| nightcam pro | trademark | Competitor brand name; complaint risk | <!-- (example — replace) -->

## Refresh Plan

- **Next scheduled review:** {{NEXT_REVIEW_DATE}} <!-- default: +90 days from {{MAP_DATE}} -->
- **Early-refresh triggers:** {{REFRESH_TRIGGERS}}
- **Refresh procedure:** re-run `01-core/prompts/13-keyword-research.md`; produce a new dated
  instance; diff quadrant changes; any title/short-description change flows through prompt 14 and
  a new `STORE_LISTING_*` instance — never edit live listings ad hoc.
- Record each refresh in `docs/factory/CHANGELOG.md`.
