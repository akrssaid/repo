# Workflow — Competitor Analysis

This workflow builds a structured competitive picture of the target app's store neighborhood:
who competes, what they say, where they are weak, and what positioning space is open. Its outputs
feed `01-core/prompts/12-aso-audit.md` (gap analysis), `01-core/prompts/14-store-assets.md`
(messaging and creative direction), and `04-aso/workflows/keyword-research.md` (seed validation
and difficulty signals). Everything here uses only freely observable store data — listings,
charts, reviews, update history — so capture dates matter: record them on every data point.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass on a target app (P7), before keyword research slotting | Full workflow |
| Quarterly refresh, paired with the keyword-map refresh | Steps 3–5 re-capture; set diff |
| A competitor visibly redesigns listing/icon or jumps in rank | Targeted re-capture of that competitor |
| Before any store-asset production cycle (`01-core/prompts/14-store-assets.md`) | Verify capture sheets are <90 days old; refresh if stale |

## Required Inputs

- `docs/factory/PROJECT_STATE.md` — the app's category, core job, and target user.
- Primary target keywords (cluster heads) from `docs/factory/reports/KEYWORD_MAP.md` if it
  exists; otherwise use the app's core-job phrase and run keyword research after.
- Store access per `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
  `04-aso/stores/rustore.md` for each store the app ships on.

## Procedure

### Stage 1 — Competitor Set Construction (target: 10 apps)

1. **Top 5 by target keyword.** Search the primary cluster-head keyword on Play (clean session,
   target locale). Take the top 5 *organic* results that are genuine functional competitors —
   skip ads and skip apps that merely keyword-match but do a different job.
2. **Top 3 category leaders.** From the app's Play category top-charts (free), take the top 3
   apps that overlap functionally. These define the ceiling: what a mature listing in this
   category looks like.
3. **2 rising apps from recent chart movement.** Inspect the category chart over 2–3 visits a few
   days apart (no paid trend tools, so movement is observed manually): pick 2 apps that are new
   to the chart, climbing between visits, or recently published (<12 months, check listing
   "Updated"/"Released" dates) with fast-growing install bands. Rising apps reveal what is
   *currently working*, which leaders' legacy equity can mask.
4. Deduplicate to a set of ~10. For AppGallery/RuStore, repeat steps 1–2 per store (smaller
   catalogs; 5–6 competitors per store is enough) — the competitive set often differs by store.

### Stage 2 — Per-Competitor Capture Sheet

5. For each competitor, complete one capture sheet (store this table per app in the output
   report). Capture **all** fields; partial sheets are not usable for synthesis.

   | Field | What to record |
   |---|---|
   | Identity | App name, package id, developer, store(s), capture date |
   | Scale | Install band, rating value, rating count |
   | Title keyword choices | Exact title; which keywords it spends title space on; brand-vs-keyword ratio |
   | Short-desc keyword choices | Exact text; keywords covered; benefit vs feature framing |
   | Icon strategy | Symbol type (literal object / abstract mark / mascot / letterform), dominant color, background style, legibility at small size |
   | First-3-screenshot messaging | Caption text of screenshots 1–3 verbatim; what benefit each leads with; device frame vs full-bleed; orientation |
   | Rating + review themes | Top recurring praise themes (most-helpful 5-star) and complaint themes |
   | 1-star review mining | Read 20–30 recent 1-star reviews; extract the 3–5 most repeated failures (crashes, ads density, paywall surprise, missing feature, privacy fears). **These weaknesses are your positioning ammo** — every repeated complaint about a leader is a claim your listing can credibly own ("no forced login", "works offline") *if the target app actually delivers it* |
   | Monetization model | Free / ads / IAP / subscription; paywall placement if discoverable; ad formats visible in screenshots or reviews |
   | Update cadence | Date of last update and apparent frequency from "What's new" history; stale (>6 months) = vulnerable |

6. **Review mining method** (for the 1-star field above, applied to at least the top 5
   competitors). **This tagged method is reusable verbatim against the TARGET app's own reviews** —
   `01-core/prompts/12-aso-audit.md` runs it on the target listing to surface self-weaknesses, and
   `04-aso/workflows/ratings-reviews.md` runs it on a cadence to route recurring themes into the
   product backlog. The labels below are the shared taxonomy; keep them stable so competitor and
   own-app minings are comparable.
   1. On the competitor's Play listing, open Ratings & reviews, filter to 1 star, sort by most
      recent; read 20–30 reviews (skip obvious spam and wrong-app reviews).
   2. Tag each review with one failure label: `crash/bug`, `ads-density`, `paywall-surprise`,
      `missing-feature`, `privacy/permissions`, `login-forced`, `performance`, `support-silence`,
      `other`.
   3. Rank labels by frequency; for the top 3–5, keep one short verbatim excerpt each (with
      review date) as evidence.
   4. Cross-check against the developer's replies: a leader that never replies to complaints is
      softer than its rating suggests; one that replies and fixes is harder to attack on that theme.
7. **Evidence discipline.** Quote review excerpts and caption text verbatim with dates. Rank and
   chart observations are point-in-time — never present them as trends unless you captured
   multiple dates.

### Stage 3 — Synthesis

8. **Build the positioning map.** Tabulate the dominant message of each competitor's title +
   screenshot 1 (e.g. "fastest", "all formats", "free", "secure", "simple"). Two columns:
   - **What everyone says** — claims made by 3+ competitors. These are table stakes; making them
     your lead message buys nothing.
   - **Gaps** — credible claims *no one* makes, and claims that map directly onto the mined
     1-star complaint themes (the strongest gaps are leader weaknesses you can prove you solve).
9. **Write the differentiation thesis** — one statement, this exact shape:
   "For [target user], [app] is the [category] that [gap-based claim], unlike [main competitors]
   which [verbatim-sourced weakness]." It must be supported by the app's real capabilities per
   `docs/factory/PROJECT_STATE.md` and by at least two cited review excerpts or capture-sheet
   facts. This thesis is the single source of truth for screenshot-1 messaging in
   `04-aso/workflows/screenshot-optimization.md` and listing copy in
   `01-core/prompts/14-store-assets.md`.

### Stage 4 — Copy vs Avoid Doctrine

10. Apply these rules when translating findings into action:
   - **Copy (conventions):** structural patterns validated by 3+ competitors — caption length and
     placement norms, screenshot count and orientation, keyword forms in titles, category-standard
     icon legibility practices. Conventions encode what converts in this category; breaking them
     needs evidence, not taste.
   - **Avoid (claims and identity):** never copy a competitor's lead differentiator (you become a
     worse version of them), brand-adjacent icon designs or color/symbol combinations
     (rejection and confusion risk — see `08-knowledge/stores/policy-landmines.md`), trademarked
     terms in title/keywords, and any claim the target app cannot demonstrably back.
   - **Exploit:** every weakness from 1-star mining that the target app genuinely solves becomes
     a candidate message, ranked by complaint frequency × the app's credibility on it.

### Stage 5 — Output & Refresh

11. Write the report to `docs/factory/reports/COMPETITOR_ANALYSIS.md` with sections: Competitor
    Set (with selection rationale), Capture Sheets (one per app), Positioning Map, Differentiation
    Thesis, Copy/Avoid/Exploit lists, Capture Dates, Next Refresh Date.
12. Update `docs/factory/PROJECT_STATE.md` (ASO section: competitor set + thesis summary) and add
    a `docs/factory/CHANGELOG.md` entry.
13. **Refresh quarterly** alongside the keyword-map refresh. On refresh, diff against the previous
    report: set changes (who entered/left), message changes (who repositioned), rating
    trajectory, and whether the differentiation thesis still holds. A thesis invalidated by a
    competitor closing the gap must be re-derived from step 7 before the next asset cycle.

## Common Failure Modes

| Failure | Symptom | Correction |
|---|---|---|
| Keyword-match competitors that do a different job | Capture sheets full of apps users would never substitute for yours | Re-apply the functional-competitor filter in step 1; replace, do not pad |
| Leader worship | Thesis copies the #1 app's message "because it works for them" | Their message works because of their scale and rating equity, not its wording — return to the gap column |
| Stale data driving an asset cycle | Capture dates >90 days old at `01-core/prompts/14-store-assets.md` time | Hard gate: refresh Stages 2–3 before any creative production |
| Thesis the app cannot back | A claim sourced from a gap, not from `docs/factory/PROJECT_STATE.md` capability | Strike it; an undeliverable claim converts installs into 1-star reviews |
| Single-store blindness | Play set reused verbatim for RuStore/AppGallery | Rebuild the keyword-derived portion per store (step 4) — catalogs differ |

## Outputs

| Artifact | Path | Consumed by |
|---|---|---|
| Competitor analysis report | `docs/factory/reports/COMPETITOR_ANALYSIS.md` | `01-core/prompts/12-aso-audit.md`, `01-core/prompts/14-store-assets.md` |
| Differentiation thesis | Section in the report, summarized in `docs/factory/PROJECT_STATE.md` | `04-aso/workflows/screenshot-optimization.md`, `04-aso/workflows/icon-optimization.md` |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` | — |

## Acceptance Criteria

- [ ] Competitor set: 5 keyword-derived + 3 category leaders + 2 rising, deduplicated, with
      stated selection rationale per app; per-store sets where the app ships on multiple stores.
- [ ] Every capture sheet 100% complete, with capture dates; no field marked unknown without a
      stated reason (e.g. store does not expose the data).
- [ ] 1-star mining done for at least the top 5 competitors, ≥20 reviews each, themes ranked by
      frequency with verbatim excerpts.
- [ ] Positioning map separates table-stakes claims from gaps.
- [ ] Differentiation thesis follows the required shape, cites ≥2 evidence items, and claims
      nothing the app cannot deliver today.
- [ ] Copy / Avoid / Exploit lists are concrete (specific patterns and claims, not generalities).
- [ ] Next refresh date recorded in the report and `docs/factory/PROJECT_STATE.md`.
