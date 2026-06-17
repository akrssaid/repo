# Play Vitals & Performance Thresholds

The **canonical numeric home** for Android vitals and runtime-performance thresholds (D10). Every
release, verification, and ASO doc that needs a vitals number — crash/ANR bad-behavior rates,
startup-time bands, frame/jank thresholds, the rollout baselines — cites **this file** rather than
restating a figure. This doc owns the *numbers and what they mean*; it does **not** own the rollout
mechanics (stage sizes, dwell times, the HOLD procedure) — those live once in
`10-releases/release-workflow.md` stage 4. It also does not own *how to measure* startup/jank — that
is `02-engineering/10-performance-audit.md`.

These are facts an agent consults from any phase, but most heavily during P4 (performance work),
P8 (verification), and post-release monitoring.

**Last reviewed: 2026-06-10.** Play's published thresholds and measurement definitions drift —
**verify against current official docs** (Play Console → Android vitals → "Core vitals" and the
"bad behavior thresholds" help pages, and the Android performance / Macrobenchmark guides) before
acting on any number below. When this file and the live Console disagree, the Console wins.

---

## 1. Why vitals matter (the gate)

Android vitals are Play's measurement of an app's technical quality from **real user devices** (the
Android vitals dataset, sampled from users who opted into sharing). Two thresholds — a **bad
behavior threshold** and a stricter **per-device** threshold — gate the app's *discovery surface*:
cross a threshold and Play reduces how often the app is recommended/surfaced and may show a warning
on the store listing for affected devices. Vitals are therefore both an engineering signal and an
ASO signal (see `08-knowledge/aso/ranking-factors.md` §5, which cites this file for the numbers).

The release workflow uses the same metrics, measured on the *current rollout cohort*, to decide
whether to advance or HOLD a staged rollout (§5 below).

---

## 2. Bad-behavior thresholds (the numbers that gate visibility)

**Verify current values in Play Console before acting.** As of the last review these are the
long-standing overall thresholds:

| Metric | Overall bad-behavior threshold | Meaning |
|---|---|---|
| **User-perceived ANR rate** | **0.47%** | % of daily-active users who experienced at least one *user-perceived* ANR (see §3). At/above this, the app is flagged in Vitals and discovery is reduced. |
| **User-perceived crash rate** | **1.09%** | % of daily-active users who experienced at least one *user-perceived* crash. Same gating consequence. |

Plus a stricter **per-device** threshold (historically around **8%** for a single high-volume
device model): an app within the overall budget can still be flagged, and warned about on that
model's listing, if one popular device is unstable. This is why instability concentrated on a
high-volume device hurts more than the same rate spread thinly across the fleet.

How to read them:

- These are **rates over daily-active users**, not raw counts. A small user base makes the rate
  noisy — one crash among few users can spike the percentage; weight conclusions by sample size.
- "At least one event" — a single user crashing repeatedly counts once toward the rate but is still
  a real defect. Triage by *affected-user* count and by clustering (one stack trace hitting many
  users beats many one-off traces).
- The thresholds are **ceilings, not targets**. The factory's internal target is materially below
  them (treat anything approaching the threshold as already failing); they exist so an app *under*
  the ceiling stays out of the penalty zone, not as a goal to ride.

---

## 3. User-perceived vs total (the distinction that matters)

Play reports both **user-perceived** and **total** variants of crashes and ANRs; the bad-behavior
thresholds above are measured on the **user-perceived** variant. The distinction:

- **User-perceived ANR**: an ANR while the app is in a state the user is actually waiting on —
  foreground, with the user interacting (an `Input dispatching timed out` on a visible activity is
  the canonical case). These are the ones that gate visibility because they reflect felt pain.
- **Total ANR**: includes background ANRs (e.g. a broadcast receiver or service exceeding its
  timeout while the user isn't watching). Real defects worth fixing, but weighted differently and
  not the primary gate.
- **User-perceived crash**: a crash of a process the user is engaged with (typically the foreground
  UI process), versus a crash in a background process the user never noticed.

Triage rule: fix **user-perceived** regressions first — they move the gating metric and the felt
quality. But do not ignore the total figures; a background ANR storm (often the
`WorkManager`/FGS/vendor-killer class — see `08-knowledge/android/common-pitfalls.md` Runtime
section) is a real reliability problem and frequently the upstream cause of the user-perceived
numbers later.

---

## 4. Performance bands (startup, frame rate, jank)

Play's "Core vitals" and the Android performance guides also surface runtime-performance metrics.
The bands below are **labeled approximations** — useful as engineering targets and review thresholds,
not exact Play penalty lines; **verify current definitions** in the performance guides. How to
*measure* each (Macrobenchmark, the Perfetto/Studio profiler, `am start -W`, JankStats / FrameTimeline)
is owned by `02-engineering/10-performance-audit.md`; this file owns only the bands.

### Startup time (time-to-initial-display)

| Launch type | Good (target) | Acceptable | Action threshold |
|---|---|---|---|
| **Cold** (process created from scratch) | **< 2 s** | < 5 s | ≥ 5 s is a defect; Play flags excessive cold start. The hardest and most-watched case. |
| **Warm** (process alive, activity recreated) | < ~1.5 s | < 2 s | Should be clearly faster than cold; if not, the activity-creation path is doing too much. |
| **Hot** (activity already resident, brought forward) | < ~1 s | < 1.5 s | Effectively instantaneous; a slow hot start means main-thread work on resume. |

Notes: measure **time-to-initial-display** (first frame) and **time-to-fully-drawn** (the app calls
`reportFullyDrawn()` when content is genuinely usable) — optimize the second, since first-frame can
be gamed with a splash. The system SplashScreen is *not* startup budget to spend; holding it on a
timer is a defect (see `08-knowledge/android/common-pitfalls.md` U5). Baseline Profiles are the
highest-leverage cold-start fix; procedure in `02-engineering/10-performance-audit.md`.

### Frame rendering — slow and frozen frames

| Band | Definition (approx.) | Target |
|---|---|---|
| **Slow / janky frame** | A frame that takes longer than its budget to render (~> 16 ms on a 60 Hz display; the budget shrinks on 90/120 Hz panels) | Keep the **share of slow frames low** — Play flags apps with an excessive slow-frame rate. |
| **Frozen frame** | A frame taking **> 700 ms** — a visible stall the user reads as "the app froze" | Drive toward **zero**; any frozen frame on a core flow is a defect. |

Jank concentrates on scroll, first interaction after launch, and transitions. Judge it on **release**
builds on **mid-range/budget hardware** — debug Compose builds are dramatically slower and will lie
to you (see `08-knowledge/android/common-pitfalls.md` U1). Recomposition storms, main-thread I/O
(StrictMode catches these — pitfall R6), and unbounded image decode are the usual causes.

---

## 5. Rollout HOLD baselines (owned by release-workflow, numbered here)

The staged-rollout **mechanics** — stage sizes (10→25→50→100%), the minimum 24h dwell per stage, and
the HOLD/halt/rollback procedure — are owned **once** by `10-releases/release-workflow.md` stage 4
(D10). Prompt 16 executes those stages by reference and does **not** keep its own table.

What this file provides is the **numeric baseline** those HOLD rules compare against, so the rollout
doc and this doc never disagree:

| Rollout guardrail | Baseline | Source of the comparison |
|---|---|---|
| Crash-rate regression HOLD | new release's user-perceived crash rate **> pre-release baseline + 0.2 pp** | release-workflow stage 4 |
| ANR-rate HOLD | new release's user-perceived ANR rate **≥ 0.40%** | release-workflow stage 4 |

These are *rollout* guardrails (a tighter, regression-sensitive bar measured on the rollout cohort),
deliberately stricter than the overall bad-behavior ceilings in §2 — the point is to catch a
regression *before* it reaches enough users to dent the published, fleet-wide rate. The "baseline" is
the prior shipped version's vitals over a comparable window. If you need the exact stage/dwell
numbers, read `10-releases/release-workflow.md` stage 4 — do not restate them elsewhere.

---

## 6. How a vitals breach routes (triage)

When a threshold (§2) or a rollout baseline (§5) is breached, the response is a method, not a number:

1. **Confirm it's real, not noise** — check the affected-user count and sample size; a tiny cohort
   spikes the rate (§2).
2. **Cluster** — Play Console / Crashlytics group by stack trace and by device/OS/API level; one
   cluster hitting many users is the priority. Symbolicate with the build's `mapping.txt` (missing
   mapping = guesswork; see `08-knowledge/android/common-pitfalls.md` RL1).
3. **Classify** — user-perceived vs total (§3); crash vs ANR; which pitfall class
   (`08-knowledge/android/common-pitfalls.md`).
4. **During a rollout** — apply the release-workflow stage-4 HOLD/halt decision against the §5
   baselines; do not advance a stage while a regression is open.
5. **Post-mortem & prevent** — feed the fix back into the engineering checklists; if it was a
   class-of-bug, add the prevention to `08-knowledge/android/common-pitfalls.md`.

---

## 7. Cross-references

- Rollout mechanics (stages, dwell, HOLD procedure): `10-releases/release-workflow.md` stage 4
- How to *measure* startup / jank / frames: `02-engineering/10-performance-audit.md`
- Vitals as a ranking/visibility signal: `08-knowledge/aso/ranking-factors.md` §5
- The pitfall classes that produce ANRs/crashes/jank: `08-knowledge/android/common-pitfalls.md`
- Mapping-file archival for symbolication: `10-releases/release-workflow.md`, pitfall RL1
- Verification report that records these numbers per release: `09-templates/verification-report.md`
- Scales (severity for vitals findings): `01-core/CONVENTIONS.md`
