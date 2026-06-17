# Workflow — Keyword Research

This workflow produces a scored, clustered, store-slotted keyword map for a target app without
paid ASO tooling. It is consumed primarily by `01-core/prompts/13-keyword-research.md` (P7) and
feeds `01-core/prompts/12-aso-audit.md` and `01-core/prompts/14-store-assets.md`. Because we do
not use paid tools (Sensor Tower, AppTweak, Mobile Action), all volume and difficulty values are
**structured estimates** derived from observable store signals — they must always be labeled
`(est.)` in outputs and never presented as measured data (conventions, ID/scale registry and the
estimate-labeling rule live in `01-core/CONVENTIONS.md`).

**This file is the canonical source for the seed and candidate volumes** referenced across the
factory: **20–40 seeds → 100–200 scored candidates**. Prompt 13 and any downstream doc cite these
numbers from here — do not redefine them elsewhere.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass on a target app (P7) | Full workflow, all stores |
| Quarterly refresh (every ~90 days) | Steps 4–9 re-run; seeds revalidated |
| Major feature ships (new capability = new query space) | Steps 1–3 for the new feature, then merge |
| Conversion or ranking drop flagged in `01-core/prompts/12-aso-audit.md` | Full re-run |
| Entering a new locale (see `04-aso/workflows/localization.md`) | Full workflow **per locale** — never translate keywords |

## Required Inputs

- `docs/factory/PROJECT_STATE.md` — app truth: actual features, category, target user.
- `docs/factory/audits/DISCOVERY.md` — feature inventory (Relevance scoring source).
- Competitor set from `04-aso/workflows/competitor-analysis.md` (or build it first).
- Store access: Google Play search (web + device), plus AppGallery / RuStore search where the
  app ships there (store rules in `04-aso/stores/`).

## Procedure

### Stage 1 — Seed Generation

1. **From app features.** List every user-facing capability from `docs/factory/audits/DISCOVERY.md`
   as a 1–3 word noun phrase (e.g. "pdf reader", "photo recovery", "video download"). Only include
   features the app actually has — Relevance scoring later depends on this being honest. Together
   with category and competitor seeds (steps 2–3), aim for the canonical **20–40 seeds** total.
2. **From category taxonomy.** Record the app's store category and subcategory; add the generic
   head terms users type for that category (e.g. Tools → "file manager", "cleaner"). Pull
   category-specific seed lists from the matching playbook in `06-playbooks/` if one exists
   (e.g. `06-playbooks/data-recovery.md`).
3. **From competitor metadata.** For each app in the competitor capture sheet
   (`04-aso/workflows/competitor-analysis.md`), extract every keyword-bearing token from title
   and short description. Tokens used by 3+ competitors are validated demand signals — add them
   as seeds even if you would not have generated them yourself.
4. **Store search-suggest harvesting.** This is the core free-volume signal. Suggestions reflect
   real query demand — the store only autocompletes what people actually type. Pick the harvest
   **mechanism** in this priority order, and record which one was used per capture (it affects
   reproducibility):
   - **Preferred — web Play with explicit locale params.** Drive the web Play search surface with
     the locale forced via URL params: `hl` = interface/language and `gl` = country/region (e.g.
     `hl=de&gl=DE` for German-in-Germany). This is repeatable, scriptable, and pins the locale
     without provisioning a device — use it whenever the web surface exposes the suggestions for
     the target store/locale.
   - **Fallback — on-device incognito Play profile.** When the web surface does not expose
     suggestions (or for AppGallery/RuStore, which have no equivalent web search), use a clean
     device or emulator: set the **device** language/region to the target locale and use an
     incognito/guest profile (no install history skewing suggestions). One profile per locale.
   - **Degradation path — operator-assisted capture.** When neither the web params nor an on-device
     profile is available to the agent (no device access, locale not provisionable), hand a capture
     brief to the human operator: the seed list, the exact prefix/letter-append steps below, and the
     fields to record. Mark every operator-supplied row `source: operator-assisted` so its
     provenance is auditable. Never fabricate suggestions to fill the gap — a missing harvest is a
     recorded limitation, not an invented signal.

   For each seed, under the chosen mechanism:
   1. Type the seed **prefix by prefix** ("pho", "phot", "photo", "photo r", "photo re" …) and
      record every autocomplete suggestion and its position (1 = top).
   2. Repeat with the seed + each letter a–z appended ("photo recovery a", "photo recovery b" …)
      to surface long-tail completions.
   3. Repeat on AppGallery and RuStore search where applicable (on-device profile path — no web
      `hl`/`gl` equivalent).
   Record per row: suggestion text, seed it came from, suggest rank, store, locale, mechanism
   used, and date.

### Stage 2 — Expansion

5. **Expand each surviving seed** along four axes; add every plausible variant to the candidate
   list:
   - **Synonyms**: "recover" / "restore" / "undelete" / "retrieve".
   - **Use-case phrasings**: "scan documents to pdf", "watch videos offline".
   - **Problem vs feature phrasings** — generate both: users search the problem
     ("recover deleted photos") far more often than the feature label ("data recovery").
     Always pair every feature phrasing with at least one problem phrasing.
   - **Long-tail combinatorics**: [modifier] × [head term] × [object/qualifier], e.g.
     {free, fast, offline, no ads} × {photo recovery} × {for android, from sd card}. Keep only
     combinations confirmed by suggest harvesting or competitor usage — do not invent traffic.
6. **Deduplicate and normalize**: lowercase, singular/plural collapsed only when the store treats
   them identically (Play partially stems; AppGallery and RuStore largely do not — keep both
   forms for those stores). Target the canonical **100–200 candidates** before scoring.

### Stage 3 — Scoring Model (all values are estimates — label them)

7. **Relevance (1–5), strictly from app truth.** Score against what the app does *today* per
   `docs/factory/PROJECT_STATE.md`, not the roadmap:
   - 5 = describes the app's core job; 4 = major secondary feature; 3 = real but minor feature;
   - 2 = adjacent (user might be satisfied, feature is partial); 1 = aspirational/misleading.
   Anything scored 1–2 is excluded from slotting. Ranking for a keyword you can't satisfy
   produces installs that churn and 1-star reviews — it is net negative.
8. **Volume (H/M/L, est.).** No paid data, so triangulate two free signals:
   - **Suggest rank**: appeared at position 1–3 for a short prefix = strong; appeared only after
     long prefixes or letter-appending = weak.
   - **Competitor-title adoption**: count of top-10 results whose *title* contains the exact
     phrase. Publishers spend title space only on terms that pay.
   | Volume (est.) | Rule |
   |---|---|
   | H | Suggest rank 1–3 on short prefix AND in 3+ top-10 titles |
   | M | Suggest appearance at any rank OR in 1–2 top-10 titles |
   | L | No suggest appearance; found only via expansion/combinatorics |
9. **Difficulty (H/M/L, est.) from top-10 result strength.** Search the exact phrase, inspect the
   top 10 results, and record:
   - **Install bands** of ranking apps (Play shows banded counts: 10K+, 1M+, 100M+ …).
   - **Title-exact-match density**: how many of the top 10 carry the exact phrase in their title.
   | Difficulty (est.) | Rule |
   |---|---|
   | H | Majority of top 10 at 10M+ installs OR 7+ exact-match titles |
   | M | Mixed bands (mostly 100K–10M) OR 3–6 exact-match titles |
   | L | Several results under 100K installs AND ≤2 exact-match titles |

### Stage 4 — Prioritization & Clustering

10. **Quadrant the scored list** (Relevance ≥4 only):
    | | Difficulty L/M (est.) | Difficulty H (est.) |
    |---|---|---|
    | **Volume H/M (est.)** | **Primary targets** — title/short-desc candidates | Long-game — full desc only; revisit when app has 100K+ installs |
    | **Volume L (est.)** | Long-tail fillers — full desc coverage | Discard |
    High-relevance/low-difficulty is the money quadrant for small and mid-size apps.
11. **Cluster into themes.** Group keywords sharing a head term or intent (e.g. "photo recovery"
    cluster: recover deleted photos, photo recovery app, restore pictures). Name each cluster,
    pick one **cluster head** (the highest Volume × lowest Difficulty member), and order clusters
    by sum of member priority. Expect 3–7 clusters.

### Stage 5 — Per-Store Slotting

12. Apply slotting rules per store (character limits and indexing rules live in
    `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
    `04-aso/stores/rustore.md`):
    - **Title** carries the strongest ranking weight on every store → exactly **1 primary
      cluster** (the cluster head phrase), integrated readably with the brand name.
    - **Short description** → **2–3 secondary clusters**, written as a benefit sentence, not a
      keyword list.
    - **Full description** → natural-language coverage of **all clusters**; apply the canonical
      keyword-density rule from `08-knowledge/aso/ranking-factors.md` (2–4 natural uses of a primary
      keyword per 4000-char description, hard cap 5). Play penalizes density abuse — do not restate
      the numbers here, cite that file.
    - Never repeat the identical phrase across title and short description to buy extra weight —
      spend the space on the next cluster instead.
13. **Instantiate the output.** Fill `09-templates/keyword-map.md` and write the instance to the
    target repo at `docs/factory/reports/KEYWORD_MAP.md`, one slotting table per store, every
    Volume/Difficulty cell suffixed `(est.)`, with harvest dates recorded.

### Stage 6 — Refresh

14. Re-run quarterly, or immediately after a major feature ships. On refresh: re-harvest suggests
    for all cluster heads (suggest sets drift), re-check top-10 strength for primary targets, and
    diff against the previous map — note movements in the instance's changelog section.

## Outputs

| Artifact | Path | Notes |
|---|---|---|
| Keyword map instance | `docs/factory/reports/KEYWORD_MAP.md` | From `09-templates/keyword-map.md` |
| Raw harvest log | `docs/factory/reports/KEYWORD_MAP.md` appendix | Suggest captures with dates/ranks |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` | Date, store(s), cluster count, primary targets |

## Acceptance Criteria

- [ ] 20–40 seeds generated; 100–200 candidates scored; every score traceable to a recorded signal
      (suggest rank, title count, install bands) with capture date.
- [ ] Every Volume and Difficulty value labeled `(est.)`; no implied measured-data claims.
- [ ] Harvest mechanism recorded per capture (web `hl`/`gl`, on-device incognito, or
      operator-assisted); no fabricated suggestions filling a harvest gap.
- [ ] No keyword with Relevance ≤2 appears in any slot.
- [ ] Exactly 1 primary cluster in each store title; 2–3 secondary in each short description;
      all clusters covered in each full description.
- [ ] Per-store slotting respects character limits in `04-aso/stores/*.md`.
- [ ] Suggest harvesting performed per target locale, not translated from English.
- [ ] Refresh date set (+90 days or next major feature) in the map instance and
      `docs/factory/PROJECT_STATE.md`.
