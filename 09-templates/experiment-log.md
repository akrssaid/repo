# Template — Experiment Log

This template produces the single running record of every store-listing experiment the factory runs
on a target app: the active experiments, the decided ones with their outcomes, and the prioritized
backlog of hypotheses waiting to run. It is the canonical log referenced by the program of record,
`04-aso/workflows/experiment-program.md` — the audit, the keyword map, the store listings, and the
asset workflows (`04-aso/workflows/icon-optimization.md`, `04-aso/workflows/screenshot-optimization.md`)
all feed hypotheses into this one file. Honesty rule: planning figures are estimates and carry an
`(est.)` label; measured experiment results (Play-reported lift and confidence) are recorded exactly
as the console reports them — the two are never blended.

## Usage

- **Instantiated by:** the ASO analyst agent. **Seeded** when the first store listing is queued for
  experiments in `01-core/prompts/14-store-assets.md`; **driven** thereafter by
  `04-aso/workflows/experiment-program.md` and the icon/screenshot asset workflows, and reviewed
  monthly via `04-aso/workflows/post-launch-monitoring.md`.
- **Phase:** P7 onward, including post-release (experiments continue across the app's life).
- **Instance path:** `docs/factory/reports/EXPERIMENT_LOG.md` in the TARGET app repo — **one
  instance per project** (a living document that grows; never re-instantiated).
- **Living document:** the Active table, Decided log, and Backlog are appended to and updated
  continuously; EXP-xxx IDs are assigned at run time and never renumbered or reused.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy the body below the
  marker, fill all tokens, delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / package id | Solar Camera Pro / com.solarcamera.pro |
| `{{LOG_OPENED_DATE}}` | Date this log was first instantiated, ISO 8601 | 2026-06-10 |
| `{{OWNER_ROLE}}` | Persona/owner running the experiment program | Senior ASO Strategist |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{STORES_IN_SCOPE}}` | Stores whose listings are under experiment | Google Play, RuStore |
| `{{ACTIVE_COUNT}}` | Count of currently running experiments | 1 |
| `{{ACTIVE_EXPERIMENT_ROWS}}` | Rows for the Active Experiments table | (table rows) |
| `{{DECIDED_EXPERIMENT_ROWS}}` | Rows for the Decided Experiments log | (table rows) |
| `{{BACKLOG_ROWS}}` | Rows for the Backlog table (queued hypotheses, ICE) | (table rows) |
| `{{LAST_REVIEW_DATE}}` / `{{NEXT_REVIEW_DATE}}` | Last / next monthly review date, ISO 8601 | 2026-06-01 / 2026-07-01 |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Experiment Log — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Log opened | {{LOG_OPENED_DATE}} |
| Program owner | {{OWNER_ROLE}} |
| Factory version | {{FACTORY_VERSION}} |
| Stores in scope | {{STORES_IN_SCOPE}} |
| Active experiments | {{ACTIVE_COUNT}} |
| Last review / next review | {{LAST_REVIEW_DATE}} / {{NEXT_REVIEW_DATE}} |

## Methodology Note

> **Estimate discipline (mandatory, keep this block):** pre-experiment traffic figures, expected
> effect sizes, and ICE inputs are **planning estimates** labeled `(est.)`. Measured results
> (Play-reported lift and confidence, or the sequential before/after delta) are recorded exactly
> as observed — never relabel a measured result as an estimate, or vice versa.

This log is governed by `04-aso/workflows/experiment-program.md` (the program of record); the rules
below are summarized from it — that file is authoritative, cite it, do not re-litigate here.

- **One live experiment per listing.** Never run two experiments (e.g. icon + screenshot) on the
  same store listing at once — the move cannot be attributed. At most one `EXP` is `Running` per
  store listing in the Active table.
- **Settle time.** After applying a winner or reverting, wait ~2 weeks before starting the next
  experiment on that listing; ranking and conversion need to re-settle.
- **Single variable.** Each experiment changes exactly one element (one surface, one variable). A
  hypothesis that names two changes is split into two backlog items.
- **Minimum runtime.** ≥7 days and until the console reports a confident result (prefer two full
  weekends); never decide on a good-looking day-2.
- **IDs:** `EXP-NNN`, zero-padded 3, per `01-core/CONVENTIONS.md` D2. An ID is assigned only when a
  backlog item is promoted to run; it is permanent thereafter.
- **Surfaces** ∈ icon / screenshot N / title / short-desc / full-desc / feature-graphic.
- **Status** uses the canonical scale (Not Started / In Progress / Blocked / Done / Deferred); a
  running experiment is In Progress, a decided one is Done.

## Active Experiments

<!-- One row per experiment currently running. Promote here from the Backlog only when it gets an
     EXP id (Stage 4 of experiment-program.md). Variable = the single element changed. Variants/arms
     = control + variant copy or a short pointer to the STORE_LISTING A/B Variants row that defines
     them (Play allows up to 3 variants vs current). Traffic split = e.g. 50/50 or 25/25/25/25.
     Status here is In Progress (or Blocked). When decided, MOVE the row to the Decided log below —
     do not leave it here. Empty is valid: write "None active as of {{LOG_OPENED_DATE}}". -->

| EXP-ID | Hypothesis | Surface | Variable | Variants / arms | Store | Start date | Traffic split | Status |
|---|---|---|---|---|---|---|---|---|
| EXP-001 | Higher-contrast icon mark lifts search CTR (est.) | icon | Icon mark contrast | Control vs high-contrast (STORE_LISTING A/B EXP-001) | Google Play | 2026-06-12 | 50/50 | In Progress | <!-- (example — replace) -->
{{ACTIVE_EXPERIMENT_ROWS}}

## Decided Experiments

<!-- The permanent outcome log — every experiment that has concluded, newest first. Result =
     Play-reported lift + confidence, or the sequential delta with its "directional only" caveat;
     include the metric delta and an estimate label ONLY where the figure is an estimate (measured
     console results are not labeled est.). Decision ∈ Adopt (apply variant) / Revert (keep control)
     / Inconclusive. Applied date = when the live listing changed (or "n/a" if reverted). Next
     action = the follow-up queued (re-test bolder, retained-installer watch, none). Rows arrive
     here by moving them out of the Active table at decision time. -->

| EXP-ID | Result (metric delta + label) | Decision | Applied date | Next action |
|---|---|---|---|---|
| EXP-002 | +6.1% install CVR, 92% confidence (Play-reported) | Adopt | 2026-06-28 | 14-day retained-installer watch; then queue screenshot-1 test | <!-- (example — replace) -->
{{DECIDED_EXPERIMENT_ROWS}}

## Backlog (queued hypotheses)

<!-- Every hypothesis not yet running, ranked by ICE. Source ∈ audit finding (ASO_AUDIT) / review
     theme / competitor diff — every item must trace to one. Hypothesis is one-variable and
     falsifiable. ICE = Impact × Confidence × Ease, each 1–10 and an estimate (label the inputs
     est.); show the three sub-scores so the rank is auditable. Items get an EXP id only when
     promoted to Active. Re-score at each monthly review — new results shift Impact/Confidence. -->

| Rank | Hypothesis | Surface | Source | Impact (est.) | Confidence (est.) | Ease (est.) | ICE (est.) |
|---|---|---|---|---|---|---|---|
| 1 | Screenshot 1 leading with "works offline" lifts CVR (review theme recurs) | screenshot 1 | Review mining | 8 | 7 | 6 | 336 | <!-- (example — replace) -->
{{BACKLOG_ROWS}}

## Maintenance Rules

- New experiments take the next free `EXP-xxx` ID; never renumber or reuse a retired ID.
- One running experiment per store listing at a time (Methodology Note); enforce the ~2-week settle
  between runs on the same listing.
- Each applied winner is recorded in `docs/factory/CHANGELOG.md` (date, EXP id, surface, decision)
  and summarized into the monthly review in `04-aso/workflows/post-launch-monitoring.md`.
- Variant copy itself lives in the `STORE_LISTING_<store>_<locale>.md` A/B Variants subsection, not
  here — this log references those rows by EXP id, it does not duplicate the copy.
