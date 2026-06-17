# ASO Post-Release Checklist

Gate run **after a release goes live**, then again on the monitoring cadence. Everything before this
gate verifies the listing *as drafted*; nothing verifies it *as published*. This checklist confirms
the store actually rendered what was uploaded, captures the post-launch performance signal against
the pre-launch baseline, keeps the review-response loop alive, watches vitals through the rollout,
starts the experiment program, and schedules the next review. A passing release that nobody watches
post-publish is how ranking regressions and policy strikes go unnoticed for weeks.

> **Run when:** After each release goes live (initial pass within 24–48h of full rollout), then on
> the monitoring cadence defined in `04-aso/workflows/post-launch-monitoring.md` (+7d, +28d).
> **Run by:** The ASO/monitoring agent, per `04-aso/workflows/post-launch-monitoring.md`.
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification.
> Record the completed copy as `docs/factory/reports/aso-post-release-YYYY-MM-DD.md`.

Scales, IDs (`EXP-NNN`, `KW-NNN`), and canonical paths per `01-core/CONVENTIONS.md`. Apply store
groups per the distribution decision in `docs/factory/PROJECT_STATE.md`.

## Listing Live-Render

- [ ] Every claimed locale spot-checked in the live store: the listing renders, all assets load,
      and there is no truncation of title / short description / what's-new in the actual store UI.
      Verify: open each store × locale listing page (or console "view as published"), confirm title,
      short/long description, what's-new, and screenshots display; record locale → OK / defect.
- [ ] All visual assets render correctly in-store: icon, feature graphic, and every screenshot
      appear at the right slot, in order, not stretched or color-shifted. Verify: compare the live
      slots against `docs/factory/assets/store/<store>/<locale>/`; record any missing or wrong asset.
- [ ] The published build/version matches the intended release: store-shown versionName and "what's
      new" correspond to this release, not a stale draft. Verify: store version vs
      `docs/factory/PROJECT_STATE.md` Release History row for this version.

## +7d Capture

- [ ] Head-term rank captured at +7d for each tracked keyword (`KW-NNN`). Verify: record the current
      store rank for each head term and compare against the Baseline Metrics Snapshot
      (head-term ranks est.) in `docs/factory/handovers/ASO_HANDOVER_v<N>.md`; note direction per term.
- [ ] Conversion rate (store-listing CVR) captured at +7d and compared against baseline. Verify:
      pull store-listing acquisition CVR by channel from the console, record vs the Baseline Metrics
      Snapshot CVR rows; flag any channel down materially as an `EXP`/backlog candidate.

## Reviews

- [ ] Review-response SLA active per `04-aso/workflows/ratings-reviews.md`: new reviews since
      launch are being replied to within the doctrine's SLA window. Verify: count reviews since the
      release date vs replies sent; confirm the oldest unanswered review is within SLA, or record
      the gap.
- [ ] New 1★ / low-rating themes triaged: recurring complaints since launch are clustered and the
      actionable ones filed to the Backlog in `docs/factory/PROJECT_STATE.md`. Verify: theme list
      attached; each actionable theme has a Backlog entry (id, severity); regression-type themes
      escalated to the engineering/design stream.

## Vitals

- [ ] Android vitals are clean after the rollout completed: user-perceived crash rate and ANR rate
      are within the bad-behavior thresholds in `08-knowledge/android/play-vitals-performance.md`
      and not regressed versus the pre-release baseline. Verify: read the post-rollout vitals in the
      console against the thresholds and startup-time bands that doc owns; record crash% / ANR% and
      pass/fail. A breach here triggers the rollback/fix-forward path in `10-releases/release-workflow.md`.

## Experiments

- [ ] The first queued listing experiment is started and logged: the `EXP-NNN` row from the ASO
      handover's EXP queue is now running in the store console and recorded in
      `docs/factory/reports/EXPERIMENT_LOG.md` (per `09-templates/experiment-log.md`). Verify:
      `Test-Path docs/factory/reports/EXPERIMENT_LOG.md` and confirm the experiment's id, hypothesis,
      variant, metric, and start date are logged; the console shows it live. N/A with justification
      if experiments are out of scope for the store in question.

## Schedule

- [ ] The +28d review is scheduled: a dated follow-up to re-run this checklist and read full-cycle
      results is recorded. Verify: the +28d date is noted in `docs/factory/PROJECT_STATE.md`
      (Open Questions or Backlog) or the monitoring tracker, consistent with
      `04-aso/workflows/post-launch-monitoring.md`.
- [ ] A KPI snapshot is written to `docs/factory/PROJECT_STATE.md`: the +7d numbers (ranks, CVR,
      installs, rating avg/count, vitals) are appended as a dated KPI snapshot so the next cycle has
      a comparison point. Verify: the snapshot block exists with this run's date and concrete values.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / FAIL |
| Stores in scope | |
| Capture window | (e.g. +7d / +28d) |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |

Log the report in `docs/factory/CHANGELOG.md`. On FAIL: file the remediation (re-upload a misrendered
asset, escalate a vitals breach to `10-releases/release-workflow.md`, start the stalled experiment),
fix, and re-run the failed group in a new dated report. This gate repeats every monitoring cadence
until the release is closed out.
