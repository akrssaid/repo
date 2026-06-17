# Workflow — Experiment Program (A/B Program of Record)

This workflow is the **program of record for every store-listing experiment** the factory runs on
a target app: how the backlog is built, how candidates are prioritized, how each run is logged with
an `EXP-NNN` id, the sequencing rules that keep results clean, and the store-by-store mechanics
(Play store-listing experiments; sequential before/after for AppGallery, RuStore, and low-traffic
apps). The asset workflows defer to this file: `04-aso/workflows/icon-optimization.md` and
`04-aso/workflows/screenshot-optimization.md` produce single-variable hypotheses and hand them
here — they never run or log experiments on their own. Conversion and A/B theory backing this
program lives in `08-knowledge/aso/conversion-optimization.md` §7; the canonical log is
`09-templates/experiment-log.md` → `docs/factory/reports/EXPERIMENT_LOG.md`. ID format, scales, and
the estimate-labeling rule are canonical in `01-core/CONVENTIONS.md` (EXP is a registered prefix in
the D2 ID registry).

> **Estimate discipline (mandatory).** Pre-experiment traffic figures, expected-effect sizes, and
> any volume/difficulty inputs to prioritization are **planning estimates** and must be labeled
> `(est.)`. Measured experiment results (Play-reported lift and confidence) are NOT estimates —
> record them as the console reports them. Never blend the two.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass on a target app (P7), once a listing is live | Stage 1 backlog build + Stage 4 first experiment (icon) |
| An asset workflow queues a hypothesis (icon, screenshot, description) | Stages 2–5 for that candidate |
| Audit finding, review-mining theme, or competitor diff suggests a listing change | Stage 1 intake → prioritize → queue |
| Post-launch monthly experiment review (`04-aso/workflows/post-launch-monitoring.md`) | Stage 5 review + re-prioritize backlog |
| An experiment completes (settle window elapsed) | Stage 5 decision + sequence the next run |

## Required Inputs

- `docs/factory/audits/ASO_AUDIT.md` — audit findings that seed hypotheses (the only ASO_AUDIT
  path; see `01-core/CONVENTIONS.md`).
- `docs/factory/reports/COMPETITOR_ANALYSIS.md` — competitor diffs and review-mined complaint
  themes (`04-aso/workflows/competitor-analysis.md`).
- `docs/factory/reports/EXPERIMENT_LOG.md` — the running log (instance of
  `09-templates/experiment-log.md`); create it from the template on first run.
- Conversion baseline(s) per surface from the asset workflows (Play Console acquisition reports).
- Store mechanics: `04-aso/stores/google-play.md` (store-listing experiments),
  `04-aso/stores/huawei-appgallery.md`, `04-aso/stores/rustore.md`.

## Procedure

### Stage 1 — Backlog Construction

1. **Source hypotheses from three streams** (every backlog item must trace to one):
   - **Audit findings** — Pass/Marginal/Fail items in `docs/factory/audits/ASO_AUDIT.md` (e.g. icon
     fails the 48 px legibility test → hypothesis: a higher-contrast mark lifts search CTR).
   - **Review mining** — recurring complaint/praise themes from the tagged taxonomy in
     `04-aso/workflows/competitor-analysis.md`, run against competitors AND the target app's own
     reviews (e.g. "works offline" recurs as a competitor gap → test it as screenshot 1's message).
   - **Competitor diffs** — positioning-map gaps and convention shifts (e.g. the shelf converged on
     blue icons → test an off-palette icon to break camouflage).
2. **Write each hypothesis in falsifiable form.** Shape:
   "Changing [variable] from [control] to [variant] will improve [metric] because [evidence],
   measured by [Play experiment result / before-after window]." One variable only — if a candidate
   names two changes, split it into two backlog items.
3. **Record the candidate** in the EXPERIMENT_LOG backlog section (not yet an EXP id): hypothesis,
   variable, surface (icon / screenshot N / short description / etc.), source stream, and the
   evidence link. Items get an `EXP-NNN` id only when they are scheduled to run (Stage 4).

### Stage 2 — ICE Prioritization

4. Score every backlog item on three axes, 1–10 each (all are estimates — label them `(est.)`):
   - **Impact (est.)** — how large a conversion/CTR move is plausible if the hypothesis holds.
     Anchor to asset leverage: icon and screenshot 1 carry the most (see
     `08-knowledge/aso/conversion-optimization.md` §3).
   - **Confidence (est.)** — strength of the evidence behind the hypothesis (a Fail-grade audit
     finding or a high-frequency review theme = high; a hunch = low).
   - **Ease (est.)** — production + test cost (a caption swap is easy; a full icon redesign is not).
5. **ICE score = Impact × Confidence × Ease (est.).** Rank the backlog by ICE. Break ties toward
   the higher-exposure surface (icon over a deep screenshot). Record each item's three sub-scores so
   the ranking is auditable and re-derivable at the next review.

### Stage 3 — Sequencing Rules (keep results clean)

6. Apply these rules without exception — most invalid ASO experiments fail here, not in the stats:
   - **One live experiment per listing.** Never run an icon and a screenshot experiment on the same
     listing at once — you cannot attribute the move. One running `EXP` per store listing at a time.
   - **Two-week settle between experiments.** After applying a winner (or reverting), wait ~2 weeks
     before starting the next experiment on that listing — ranking and conversion need time to
     re-settle (`08-knowledge/aso/ranking-factors.md` §8).
   - **Seasonality blackouts.** Do not start or conclude an experiment across holidays, major sales
     periods, a featuring spike, or a known LiveOps moment — the cohort is not representative
     (`08-knowledge/aso/conversion-optimization.md` §7; calendar in
     `04-aso/workflows/post-launch-monitoring.md`).
   - **Minimum runtime.** ≥7 days **and** until the console reports a confident result — prefer two
     full weekends (weekday/weekend traffic differs in quality). Never stop on a good-looking day-2.

### Stage 4 — Running an Experiment

7. **Assign the EXP id and open a log row.** Promote the top backlog item: give it the next
   `EXP-NNN` id and create its row in `docs/factory/reports/EXPERIMENT_LOG.md` with the canonical
   fields from `09-templates/experiment-log.md`: **id, hypothesis, variable, variant(s), start
   date, planned end, arms/traffic split, store, status** (status uses the
   `01-core/CONVENTIONS.md` scale; result/decision/next-action are filled at Stage 5).
8. **Play — store listing experiments.** Configure in Play Console → Store listing experiments with
   exactly one element changed. Play supports **up to 3 variants vs the current listing** (per D16;
   mechanics in `04-aso/stores/google-play.md` — that file is authoritative for console steps).
   Default split is 50/50 (1 variant vs control); use 3 variants only when the listing has the
   traffic to power three arms. Below ~1,000 installs per arm per week, asset experiments are noise
   — fall back to the sequential method (step 9).
9. **AppGallery, RuStore, and low-traffic apps — sequential before/after.** These stores have no
   native A/B tooling, and low-traffic Play listings cannot power a split. Method:
   1. Record the baseline metric (store-listing conversion / CTR) over a clean window, typically
      4 weeks, away from any seasonality.
   2. Apply the change to the live listing.
   3. Measure the same metric over an equal post-change window.
   4. Treat the delta as **directional, not significant** — confounded by time, seasonality, and
      concurrent changes. Require a large, durable move before crediting the change; hold all other
      listing variables constant across both windows.
10. **What to test first.** Order the first experiments by exposure and effect size:
    **icon first** (it appears at every surface — search, charts, rails, launcher), **then
    screenshot 1's message**. Description and deeper screenshots follow only after the
    highest-leverage assets are settled.

### Stage 5 — Decision, Logging, and Re-Sequencing

11. **Decide only on reported significance.** Apply a variant when the console reports a confident
    positive result on installs; keep control when confidently negative or inconclusive. An
    inconclusive result on a high-leverage asset usually means the variable was too timid — re-queue
    a bolder variant rather than re-running the same one.
12. **Post-apply watch.** After applying any winner, watch a 14-day window for a decline in retained
    installers (Play reports retained-installer conversion). A change that buys taps from the wrong
    users shows up here — revert if retained-install conversion drops.
13. **Close the log row.** Fill **result** (Play-reported lift + confidence, or the sequential
    delta with its caveat), **decision** (Applied / Kept control / Reverted), and **next action**
    in `docs/factory/reports/EXPERIMENT_LOG.md`. Set status to Done.
14. **Re-sequence.** After the 2-week settle, re-score the backlog (Stage 2 — new findings and the
    just-learned result shift Impact/Confidence) and promote the next item. Feed the result summary
    to the monthly experiment review in `04-aso/workflows/post-launch-monitoring.md`.

## Outputs

| Artifact | Path | Notes |
|---|---|---|
| Experiment log (backlog + EXP rows + results) | `docs/factory/reports/EXPERIMENT_LOG.md` | From `09-templates/experiment-log.md` |
| EXP-queue rows for handover | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` (EXP-queue section) | IDs consumed by the log (D16) |
| Experiment outcome summary | Monthly review in `04-aso/workflows/post-launch-monitoring.md` | Feeds KPI snapshot |
| CHANGELOG entry per applied winner | `docs/factory/CHANGELOG.md` | Date, EXP id, surface, decision |

## Acceptance Criteria

- [ ] Every backlog item traces to an audit finding, a review-mined theme, or a competitor diff,
      and is written as a one-variable falsifiable hypothesis.
- [ ] Each item carries Impact/Confidence/Ease sub-scores (`(est.)`) and an ICE rank.
- [ ] Sequencing rules enforced: one live experiment per listing, ≥2-week settle, no run across a
      seasonality blackout, ≥7-day minimum runtime.
- [ ] Each running experiment has an `EXP-NNN` id and a complete log row in
      `docs/factory/reports/EXPERIMENT_LOG.md` (hypothesis, variable, variant, dates, arms/traffic,
      result, decision, next action).
- [ ] Play experiments use ≤3 variants vs current; low-traffic / AppGallery / RuStore use the
      sequential before/after method with its directional-only caveat stated.
- [ ] First experiments ordered icon → screenshot 1; decisions made only on reported significance;
      post-apply retained-installer watch done for applied winners.
- [ ] No measured result presented as an estimate and no estimate presented as measured.
