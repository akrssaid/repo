# Workflow — Post-Launch ASO Monitoring

The factory currently goes quiet after launch; this workflow is the recurring cadence that keeps
the listing alive. It defines weekly, monthly, and quarterly cycles, the KPI snapshot written to
`PROJECT_STATE.md` each cycle, a trigger table routing anomalies to their owner prompts, and a
seasonal/featuring calendar. It runs against `07-checklists/aso-post-release.md` as the recurring
gate (that checklist is the pass/fail wrapper; this workflow is the method). Conventions, scales,
and canonical paths are in `01-core/CONVENTIONS.md`; conversion benchmarks (all estimates) live in
`08-knowledge/aso/conversion-optimization.md` §8.

> **Estimate discipline.** Benchmark bands and any projected trend are **estimates** — label
> `(est.)`. Measured console figures (vitals, conversion, rating, ranks where observed) are
> recorded as measured. Keyword ranks gathered by manual spot-check are observations, not measured
> rank-tracking data — note the method.

## When to Run

| Trigger | Scope |
|---|---|
| First production release shipped (`01-core/prompts/16-release-preparation.md` complete) | Start the cadence; set the weekly/monthly/quarterly clock |
| Each weekly cycle | Stage 1 |
| Each monthly cycle | Stages 1 + 2 |
| Each quarterly cycle | Stages 1 + 2 + 3 (full refresh) |
| Any anomaly threshold crossed | Stage 4 trigger routing (out of band) |

## Required Inputs

- Console access: Play Console (vitals, acquisition, ratings), AppGallery Connect, RuStore console.
- `docs/factory/PROJECT_STATE.md` — KPI snapshots land here; read the **Artifact Index** and
  **Release History** sections (D6 / `01-core/CONVENTIONS.md`) for before/after comparability.
- `docs/factory/audits/ASO_AUDIT.md`, `docs/factory/reports/KEYWORD_MAP.md`,
  `docs/factory/reports/COMPETITOR_ANALYSIS.md`, `docs/factory/reports/EXPERIMENT_LOG.md`.
- Vitals thresholds: `08-knowledge/android/play-vitals-performance.md` (bad-behavior thresholds +
  startup bands; D10). Rollout/vitals gate numbers live in `10-releases/release-workflow.md`.

## Procedure

### Stage 1 — Weekly Cycle

1. **Vitals.** Check Play Console → Android vitals against the thresholds in
   `08-knowledge/android/play-vitals-performance.md`: crash rate, ANR rate, and the bad-behavior
   thresholds (1.09% / 0.47%). A breach is a release-health issue, not an ASO one — route per the
   trigger table (Stage 4) and to `10-releases/release-workflow.md` if a rollout is in progress.
2. **Rating recency.** Read the last 7 days of new ratings/reviews; note the recent-window average
   and any new low-rating cluster. Recency drives the displayed rating and conversion.
3. **Review reply queue.** Clear the SLA queue per `04-aso/workflows/ratings-reviews.md`: every
   1–2★ replied within 72h. Triage new reviews to the shared failure taxonomy; escalate any
   recurring product-bug theme to `docs/factory/PROJECT_STATE.md` Open Questions / Backlog.

### Stage 2 — Monthly Cycle

4. **Conversion by channel.** Pull Play Console acquisition: store-listing conversion split by
   search vs browse, last 28 days, vs the prior month and the `(est.)` benchmark bands in
   `08-knowledge/aso/conversion-optimization.md` §8. Always segment by channel before reacting — a
   blended drop after a featuring spike is usually mix-shift, not asset decay.
5. **Keyword-rank spot checks.** For each primary cluster head in
   `docs/factory/reports/KEYWORD_MAP.md`, manually search the term in the target locale (clean
   session) and record the app's approximate position. This is an **observation**, point-in-time —
   never present it as tracked rank data; capture the date and method.
6. **Experiment review.** Review running and recently-closed experiments in
   `docs/factory/reports/EXPERIMENT_LOG.md` with `04-aso/workflows/experiment-program.md`: close
   settled experiments, re-score the backlog, and (respecting the 2-week settle and seasonality
   blackouts) promote the next experiment.

### Stage 3 — Quarterly Cycle

7. **Full keyword refresh.** Re-run `04-aso/workflows/keyword-research.md` (re-harvest suggests for
   all cluster heads, re-check top-10 strength, diff against the previous `KEYWORD_MAP.md`). Push
   any title/short-description change through `01-core/prompts/14-store-assets.md` — never edit live
   listings ad hoc.
8. **Full competitor refresh.** Re-run `04-aso/workflows/competitor-analysis.md` (set changes,
   repositioning, rating trajectory, thesis still-valid check). A thesis invalidated by a competitor
   closing the gap is re-derived before the next asset cycle.
9. **Localization review.** Run `04-aso/workflows/localization.md` Stage 7: per-locale conversion
   deltas; promote converting locales, freeze flat ones.

### Stage 4 — KPI Snapshot to PROJECT_STATE

10. Each cycle, write a dated KPI snapshot row so cycles are comparable over time. Record it in
    `docs/factory/PROJECT_STATE.md`, tied to the **Release History** entry (versionName/code) and
    cross-referenced from the **Artifact Index** (D6 sections — `01-core/CONVENTIONS.md`) so a
    reader can line each snapshot up against the release and artifacts that produced it.

    | Field | Source | Note |
    |---|---|---|
    | Date / cycle | this run | weekly / monthly / quarterly |
    | versionName / code | Release History | comparability anchor |
    | Crash-free % / ANR % | Play vitals | vs threshold |
    | Rating avg / count / recent-window avg | console | recency matters |
    | Conversion (search / browse) | acquisition | vs prior + benchmark `(est.)` |
    | Primary cluster ranks (observed) | spot check | point-in-time, dated |
    | Open experiments / last result | EXPERIMENT_LOG | EXP ids |

### Stage 5 — Anomaly Trigger Routing

11. When a KPI crosses a threshold, route it to the **owner** rather than fixing it ad hoc here:

    | Anomaly | Threshold (est. — calibrate) | Owner |
    |---|---|---|
    | Listing conversion drop | down materially vs prior month, not mix-shift | `01-core/prompts/12-aso-audit.md` (re-audit assets) |
    | Keyword-rank drop on a primary head | fell out of the top results it held | `01-core/prompts/13-keyword-research.md` (`04-aso/workflows/keyword-research.md`) |
    | Rating drop / low-rating cluster | recent-window avg falling | `04-aso/workflows/ratings-reviews.md` (recovery program) |
    | Crash / ANR breach | over `play-vitals-performance.md` thresholds | engineering + `10-releases/release-workflow.md` (HOLD if rolling out) |
    | Competitor reposition / new entrant | thesis at risk | `04-aso/workflows/competitor-analysis.md` |
    | Data-safety mismatch (dependency change shipped) | any new SDK / data flow | `04-aso/workflows/data-safety-listing.md` |

### Stage 6 — Seasonal & Featuring Calendar

12. Maintain a forward calendar so experiments avoid blackouts and the team plans creative around
    high-traffic moments:
    - **Play** — LiveOps moments (events, offers, major updates surfaced on the store) and seasonal
      featuring windows; plan listing refreshes and screenshot seasonals around them, and **blackout
      experiments** during featuring spikes (cohort not representative).
    - **AppGallery** — Huawei campaign seasons / editorial submission windows; submit campaign
      assets ahead of the season.
    - **RuStore** — RuStore collections / editorial placements; align RU creative to collection
      themes.
    Record upcoming moments and their experiment-blackout dates; share with
    `04-aso/workflows/experiment-program.md` (Stage 3 seasonality blackouts).

## Common Failure Modes

| Failure | Symptom | Correction |
|---|---|---|
| Cadence lapses after launch | No KPI snapshot for weeks; reply queue stale | Treat the weekly cycle as a standing gate via `07-checklists/aso-post-release.md`, not optional |
| Reacting to a blended drop | "Conversion fell" with no channel split | Segment search vs browse first (Stage 2 step 4); featuring spikes cause mix-shift, not decay |
| Spot-check ranks read as tracking | A point-in-time observation cited as a trend | Always date the observation and mark it point-in-time; need ≥2 dated checks before claiming movement |
| Anomaly fixed ad hoc here | A rank/rating dip "handled" without the owner workflow | Route via the Stage 4 trigger table to the owning prompt/workflow |
| Experiment started in a blackout | Test launched into a featuring/seasonal spike | Check the Stage 6 calendar before promoting any experiment |

## Outputs

| Artifact | Path | Notes |
|---|---|---|
| KPI snapshot (per cycle) | `docs/factory/PROJECT_STATE.md` | Tied to Release History; comparable over time |
| Updated audit / maps on refresh | `docs/factory/audits/ASO_AUDIT.md`, `reports/KEYWORD_MAP.md`, `reports/COMPETITOR_ANALYSIS.md` | Quarterly |
| Anomaly routing record | `docs/factory/PROJECT_STATE.md` (Open Questions / Backlog) | With owner prompt |
| Seasonal/featuring calendar | `docs/factory/PROJECT_STATE.md` or `docs/factory/audits/ASO_AUDIT.md` | Feeds experiment blackouts |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` | Per refresh |

## Acceptance Criteria

- [ ] Cadence running: weekly (vitals, rating recency, reply queue), monthly (conversion by channel,
      keyword-rank spot checks, experiment review), quarterly (full keyword + competitor + locale
      refresh).
- [ ] A dated KPI snapshot written to `docs/factory/PROJECT_STATE.md` each cycle, anchored to the
      Release History version for before/after comparability; benchmarks labeled `(est.)`, observed
      ranks dated and marked point-in-time.
- [ ] Anomalies routed via the trigger table to their owner prompt/workflow, not fixed ad hoc.
- [ ] Seasonal/featuring calendar maintained and shared with the experiment program's blackouts.
- [ ] `07-checklists/aso-post-release.md` run as the recurring gate each cycle.
