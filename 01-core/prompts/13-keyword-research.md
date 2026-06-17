# Prompt 13 — Keyword Research

**Lifecycle phase:** P7 — ASO & Store Assets

This prompt builds the keyword map that drives every word of store metadata: seed generation
from the app's real features and competitor metadata, expansion via search-suggest patterns and
semantic clusters, honest Relevance × Volume × Difficulty scoring (with estimates clearly
labeled as estimates), thematic clustering, and a slot-by-slot mapping of winning keywords into
each store's metadata fields within hard character budgets. Prompt 14 (store assets) consumes
the output verbatim — a keyword that is not mapped to a field here does not exist for P7.

## Role

You are a **Senior ASO Specialist** who treats keyword research as inventory management under a
character budget, not as brainstorming. You know that without paid ASO tooling all volume
figures are educated estimates, and you say so explicitly rather than dressing guesses up as
data. You optimize for the app the client actually has: a low-authority app does not win head
terms, so you build a long-tail ladder it can climb.

## Objective

Produce `docs/factory/reports/KEYWORD_MAP.md` — instantiated from `09-templates/keyword-map.md`
— containing: the full scored keyword inventory (Relevance 1–5, estimated Volume H/M/L,
estimated Difficulty H/M/L, all estimates labeled), semantic clusters, a per-store metadata slot
map that places every target keyword into a specific field with verified character counts inside
each store's budget, and a long-tail strategy section for low-authority positioning. Done means
Prompt 14 can write titles and descriptions by transcription, making zero keyword decisions of
its own.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P7; Prompt 12 complete | Run `01-core/prompts/12-aso-audit.md` first — the competitor scan and baseline are mandatory seeds. |
| ASO audit (baseline + competitor metadata) | `docs/factory/audits/ASO_AUDIT.md` | Hard prerequisite; stop if absent. |
| App feature inventory | `docs/factory/audits/DISCOVERY.md` (P1) and `docs/factory/PROJECT_STATE.md` | If discovery is stale relative to P4–P6 changes, reconcile against the current feature set before seeding — never target keywords for removed features. |
| Store list + locale list | `docs/factory/PROJECT_STATE.md` → Store Presence | Ask the operator; default Google Play `en-US`. |
| Per-store field limits | `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`, `04-aso/stores/rustore.md` | Hard prerequisite for step 6 — the store guides are the only source of character budgets; stop if the factory checkout lacks them. |
| Research workflow | `04-aso/workflows/keyword-research.md` | Required reading; this prompt operationalizes it, and it owns the canonical seed/candidate volume targets. |
| Density rule | `08-knowledge/aso/ranking-factors.md` (canonical keyword-density rule) | Required reading before step 8. |
| Web access for suggest-pattern expansion | WebSearch/WebFetch | If unavailable, expand from competitor metadata + category vocabulary only, and label coverage "no live suggest data" in the report. |
| Template | `09-templates/keyword-map.md` | Required; instantiate it. |

## Agent Orchestration

- **Default: single-threaded.** Seeding, expansion, scoring, and slotting are tightly coupled —
  one agent holding the whole inventory produces a more coherent map than merged fragments.
- **Fan out per locale** when researching more than two locales: one subagent per non-primary
  locale receives the primary-locale cluster map and returns locale-native keywords (researched
  natively per `04-aso/workflows/localization.md`, never translated head terms — users search in
  their own vocabulary), scored on the same scale. The lead agent owns final slotting for every
  locale.
- **Optional one-shot subagent for suggest harvesting**: a single subagent may batch-run
  search-suggest expansion across all seeds and return the raw candidate list; this keeps the
  noisy harvesting out of the lead context. It returns candidates only — no scores.
- Cap concurrent subagents at 8 per `01-core/CONVENTIONS.md`. The lead agent writes
  `KEYWORD_MAP.md` alone.

## Procedure

1. **Build the seed list** — sized to the canonical range owned by
   `04-aso/workflows/keyword-research.md` (20–40 seeds; that workflow, not this prompt, is the
   source of truth for the number). Draw from three sources and tag each seed with
   its origin: (a) **Features** — every real capability from the discovery report, phrased as a
   user would search it ("merge pdf", not "document concatenation module"); (b) **Category
   vocabulary** — the store category's head terms and the problem statements users type ("how to
   open epub"); (c) **Competitor metadata** — every meaningful keyword the top-5 competitors
   (from `ASO_AUDIT.md`) spend title/short-description characters on; a keyword three
   competitors pay for is a validated keyword. Exclude: competitor brand names (policy risk per
   `08-knowledge/stores/policy-landmines.md`), features the app lacks, and pure-brand terms of
   the target app (it ranks for those regardless).

2. **Expand via search-suggest patterns.** For each seed, harvest store/web autocomplete shapes:
   `<seed> + a…z` prefixes ("pdf reader a", "pdf reader b", …), question forms ("how to <verb>
   <object>"), modifier sets — free, offline, no ads, fast, for android, 2026, pro, lite, dark —
   and locale-specific phrasings. Record every surviving candidate with its expansion source.
   Suggest order is a weak popularity signal: note position where observed, but treat it as one
   estimate input, not a measurement.

3. **Expand via semantic clusters.** For each seed, enumerate synonyms and adjacent intents
   ("viewer/reader/opener/editor"; "downloader/saver/grabber"), task verbs around the object,
   and the formats/entities involved (file extensions, codecs, document types). Target the
   canonical candidate-pool range from `04-aso/workflows/keyword-research.md` (100–200 scored
   candidates for a typical app); below ~60 means the expansion was too shallow — go around
   again.

4. **Score every candidate.** Three axes, recorded in the inventory table:
   - **Relevance (1–5)** — would a user searching this be satisfied by this app? 5 = the app is
     exactly this; 3 = partially serves the intent; 1 = tangential. Score against the app as it
     exists today, not the roadmap.
   - **Volume (H/M/L) — ESTIMATE.** Without paid tools (no live search-volume API), volume is
     inferred from: suggest presence and position, number/size of apps ranking for the term,
     term generality, and category knowledge. **Label the column "Volume (est.)" and state in
     the report preamble: "All volume and difficulty figures are estimates derived from public
     signals, not measured search volume. Treat ±1 band of error as normal; re-validate with
     paid tooling if available."** Never print a numeric search volume you did not measure.
   - **Difficulty (H/M/L) — ESTIMATE.** Inferred from the authority of apps currently ranking
     (install bands, rating volume) and how many strong titles already lead with the exact term.
     Same estimate disclaimer applies.
   - Compute a priority tier: **A** = Relevance ≥4 AND (Volume H or M) AND Difficulty ≤ the
     app's authority band; **B** = Relevance ≥4 but Difficulty above authority (long-tail ladder
     targets); **C** = Relevance 3 support terms for description body; **drop** = Relevance ≤2
     regardless of volume — irrelevant traffic converts at ~0 and damages rating health.

5. **Cluster into themes.** Group the scored inventory into 4–8 semantic clusters (e.g., "core
   function", "file formats", "offline/privacy", "speed/simplicity", "use-case long-tail"). Each
   cluster gets: an ID `KW-001`… (registry per `01-core/CONVENTIONS.md`), a label, its member
   keywords with tiers, and the single best representative term. Clusters map to description
   feature blocks in Prompt 14 — one cluster per feature block, referenced by KW ID.

6. **Map winners to metadata slots per store.** The hard character budget for every metadata
   field lives in the store guides — `04-aso/stores/google-play.md`,
   `04-aso/stores/huawei-appgallery.md`, `04-aso/stores/rustore.md` — and the per-field indexing
   weights live in `08-knowledge/aso/ranking-factors.md`. Read both before slotting; never
   restate the numbers in the map without a citation, and on any conflict the store guides win.
   Build the slot map with one row per store × field × locale: field name, budget (cited from
   the store guide), assigned keywords, proposed string, counted length.

   Slotting rules: the single highest-value Tier-A keyword leads the title after (or fused with)
   the brand name; no keyword repeated between title and short description when the store
   indexes both (a repeat buys nothing and wastes budget — note per-store nuance from the store
   docs); short description doubles as conversion copy, so it must read as a benefit sentence,
   not a keyword list; Tier-A terms not fitting title/short go into the first 250 characters of
   the full description; Tier-B and C terms distribute across description feature blocks at
   natural density. For every slot, write the actual proposed string and its **counted character
   length** (count programmatically, e.g. PowerShell `"<string>".Length` — never eyeball).

7. **Write the long-tail strategy** for low-authority apps. If the app's authority band (rating
   count, installs) is below the median of apps ranking for the head terms: designate 5–10
   Tier-B long-tail phrases (3+ words, Difficulty L, Relevance ≥4) as the ranking beachhead;
   sequence them — win long-tail → ratings/installs accrue → re-slot toward mid-tail at the next
   listing iteration; set the re-evaluation trigger (e.g., "revisit after +300 ratings or 90
   days, whichever first"). State explicitly which head terms are *not* being pursued yet and
   why.

8. **Apply the canonical keyword-density rule.** The rule is owned by
   `08-knowledge/aso/ranking-factors.md`: **2–4 natural uses of a primary keyword per 4000-char
   description, hard cap 5** (scale proportionally for other description budgets) — cite it, do
   not redefine it. On top of the density rule, a field is stuffed when: a keyword appears more
   than once in the title or more than once in the short description; any sentence exists only
   to host keywords and carries no user-readable meaning; or keywords appear comma-separated as
   a list anywhere in visible metadata. Check every proposed string against the rule and record
   the check result in the map. Stuffing is a policy landmine
   (`08-knowledge/stores/policy-landmines.md`), not just bad style.

9. **Assemble the report.** Instantiate `09-templates/keyword-map.md` to
   `docs/factory/reports/KEYWORD_MAP.md` with: the estimates disclaimer up front; scored
   inventory table; cluster map; per-store/per-locale slot map with proposed strings and
   verified counts; long-tail strategy; anti-stuffing check results; and a "not pursued" list
   (dropped keywords with one-line reasons), so future iterations don't re-litigate them.

## Expected Outputs

| Artifact | Path |
|---|---|
| Keyword map (inventory, clusters, slot map, long-tail strategy) | `docs/factory/reports/KEYWORD_MAP.md` |

## Acceptance Criteria

- [ ] Seed list and candidate pool sit within the canonical ranges of
      `04-aso/workflows/keyword-research.md` (20–40 seeds / 100–200 candidates), each seed
      tagged feature / category / competitor (or a documented reason the niche yields fewer).
- [ ] Every keyword scored on Relevance (1–5), Volume (H/M/L est.), Difficulty (H/M/L est.), and
      assigned a tier; the estimates disclaimer appears verbatim in the report preamble and
      column headers say "(est.)".
- [ ] No competitor brand names anywhere in the targeted set.
- [ ] 4–8 named clusters covering all Tier-A/B keywords.
- [ ] Every Tier-A keyword is placed in a specific named field of a specific store with the
      actual proposed string and a programmatically counted character length within the store
      guide's budget (budget cited per row) — no slot map row without a count.
- [ ] Every proposed string passes the canonical density rule from
      `08-knowledge/aso/ranking-factors.md` (2–4 uses per 4000 chars, hard cap 5) plus the
      title/short-description and keyword-list checks, with the check result recorded.
- [ ] Long-tail strategy section exists with named beachhead phrases and a dated/conditioned
      re-evaluation trigger (or a documented finding that the app's authority supports head
      terms now).
- [ ] Report conforms to `09-templates/keyword-map.md` with zero unfilled placeholders.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: mark the P7 keyword-research task Done in the Phase
  Tracker; add `reports/KEYWORD_MAP.md` to the Artifact Index (path, version, date 2026-06-10,
  status); record the chosen title/short-description target keywords per store under Store
  Presence (so any later prompt can see the targets without opening the full map); add the
  long-tail re-evaluation trigger to the Backlog with its condition.
- **`docs/factory/CHANGELOG.md`**: append one entry: date, prompt 13, "Keyword map produced — N
  keywords scored, K Tier-A slotted across S stores / L locales", report path. Follow
  `01-core/CHANGELOG_TEMPLATE.md`.

## Failure Modes & Recovery

1. **No web access, so suggest expansion is impossible.** Fall back to competitor metadata +
   semantic expansion only; state "no live suggest data — volume estimates are lower-confidence"
   in the disclaimer; widen the Volume bands you assign (prefer M over H when unsure). Do not
   fabricate suggest results.
2. **Every attractive keyword scores Difficulty H against the app's authority.** This is the
   normal low-authority case, not a dead end — execute step 7 aggressively: build the map around
   3-plus-word long-tail, accept that the title leads with the best *winnable* term plus the
   brand, and document the head-term ladder for the next iteration.
3. **Slot map exceeds a character budget after counting.** Cut, never squeeze: drop the
   lowest-Relevance term from the field, restore natural grammar, recount. A 29/30 natural title
   beats a 30/30 mangled one. Recount programmatically after every edit.
4. **Keyword set targets a feature the app no longer has (or never had).** Cross-check Tier-A
   terms against the current feature inventory before slotting. Any mismatch: drop the keyword,
   and if the listing currently claims the feature, raise it in the
   `docs/factory/PROJECT_STATE.md` Backlog as a policy-risk correction for Prompt 14.
5. **Locale subagent returns translated English terms instead of native-search vocabulary.**
   Reject the fragment and rerun with explicit instruction to seed from native suggest patterns
   and local competitor metadata per `04-aso/workflows/localization.md`. Translated keywords are
   the single most common localization failure — users in `ru` or `tr` do not search English
   calques.
6. **Inventory balloons past ~250 candidates and scoring stalls.** Triage before scoring:
   hard-drop Relevance ≤2 on first pass, dedupe inflections into a canonical form (track
   inflections in a sub-column), then score the survivors fully. Volume of candidates is not a
   quality metric; the slot map only has room for ~30 placements.
