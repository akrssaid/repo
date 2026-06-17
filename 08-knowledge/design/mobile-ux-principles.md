# Mobile UX Principles for Utility Apps

Evergreen UX doctrine for the factory's portfolio: utility apps (file managers, document readers,
downloaders, recovery tools) where users arrive with a job, not a desire to explore. These
principles are the evaluation rubric for design audits (`03-design/01-design-audit.md`) and the
design constitution for redesigns (P5–P6). Unlike platform facts, this doctrine is stable — only
the predictive-back and ad-policy touchpoints drift; **verify against current official docs** for
those specifics.

**last reviewed: 2026-06-10**

---

## 1. Core-Task-First

A utility app is hired for one job. The entire UI bends toward completing that job fast.

- **The 3-tap rule**: from cold launch, the user reaches primary value in ≤3 taps. Open file
  manager → see files → open file: 2 taps. Audits count taps for the top-3 jobs and log every
  interstitial (splash delay, rating prompt, upsell, "what's new") standing between launch and
  the job — each is a defect with severity proportional to frequency of the blocked task.
- **Home screen = the job, not a menu.** The first screen shows the work surface (the files, the
  documents, the scan button), not a grid of feature tiles. A launcher-of-features home screen is
  the signature of an app designed around its own org chart.
- The most recent/likely object of the job appears first (recent files, last session, resume
  point). Continuing beats starting over.

## 2. Progressive Disclosure

- Advanced features **earn their place**: ship them behind "More", long-press, or settings until
  usage data argues for promotion. Every always-visible control taxes every user on every visit.
- Defaults do the right thing without configuration; settings exist for the 5% who need to
  deviate, and the app never *requires* a settings visit to complete the core job.
- Disclosure is layered, not hidden: primary action visible → secondary one gesture away →
  tertiary in overflow/settings. "Mystery meat" (see anti-patterns) is hiding, not disclosure.

## 3. State Completeness

Every screen is designed in **all five states** of the per-screen states contract, not one. The
canonical state set — `default / empty / loading / error / offline` (plus dark for every state) — is
defined once in `01-core/CONVENTIONS.md`; the design audit (`03-design/01-design-audit.md`), the
screen inventory, and the DESIGN handover's `States present` column all use exactly these names.
Audits check all five; handovers (`05-handover/DESIGN_HANDOVER_SCHEMA.md`) specify them per screen.

| State (per `01-core/CONVENTIONS.md`) | Requirement |
|---|---|
| **default** | The happy path with realistic data volume (50 files, not 3) — the screen doing its job |
| **empty** | **Teaches**: states what belongs here, why it is useful, and offers the one action that fills it. Never a bare "No items" |
| **loading** | Skeletons/placeholders matching final layout (no layout jump); spinner only for sub-second unknowns |
| **error** | Says what failed *in user terms*, what the app already tried, and gives a next step (retry, settings link, support). Never a raw exception or bare "Something went wrong" — copy patterns in `08-knowledge/design/ux-writing-essentials.md` |
| **offline** | Distinguished from error (offline is a recoverable condition, not a failure): cached content remains usable and labeled; queued actions visible. Now a first-class member of the contract, not an optional sixth state |

The earlier "ideal" label for the happy-path state is retired in favor of the canonical **default**;
sweep any per-project artifact still naming it "ideal".

## 4. Feedback Latency Rules

| Budget | Rule |
|---|---|
| < 100 ms | Every touch visibly acknowledged (ripple, state change) — perceived as instantaneous |
| < 1 s | Operations finishing under ~1s need no progress UI, but show result confirmation |
| > 1 s | Determinate progress where possible, plus **cancel**; indeterminate spinner acceptable only when truly unknowable |
| > 10 s | Progress with remaining estimate or count ("142/580 files"), survives screen-off and backgrounding (see `08-knowledge/android/modern-stack.md` background table), and reports completion via notification when appropriate |

Never freeze the UI: long work moves off the main thread (an ANR is a UX failure first, a
technical defect second). Never lie with fake progress bars — users learn the lie within two uses.

## 5. Forgiveness

- **Undo over confirm.** Confirmation dialogs train blind tapping; undo (snackbar with action,
  trash with retention window) keeps users fast *and* safe. Reserve confirmation dialogs for
  irreversible, high-stakes actions (permanent delete of 500 files; format) — and make those
  rare by making more actions reversible.
- Destructive actions are recoverable by design: soft-delete with a visible trash, export before
  overwrite, "restore previous" after batch operations.
- Errors of navigation are free: back always available, partial form input preserved, no "are you
  sure you want to leave" guilt dialogs for read-only screens.

## 6. Navigation Honesty

- **Back always works predictably**: one step backward in the user's mental journey, never exits
  unexpectedly, never loops, never re-triggers a completed action or re-shows a dismissed
  interstitial.
- **Predictive back implications** (Android 14+, default at targetSdk 36 — see
  `08-knowledge/android/version-matrix.md`): users *peek* at the back destination before
  committing. Any screen that hijacks back (custom handlers for "press again to exit", ad
  interstitials on back, save prompts) must integrate with the predictive gesture or the peek
  reveals a lie. Audit every `OnBackPressedCallback`.
- Up/back distinction, deep-link entries, and notification taps all reconstruct a sane back
  stack — landing somewhere with no way back but app-exit is a defect.

## 7. Onboarding Doctrine for Utility Apps

- **Skip tours. Teach in context.** Multi-page intro carousels are skipped by ~everyone and delay
  the job (violating §1). The first run goes straight to the work surface; novel UI gets a
  one-time inline hint at the moment of first relevance.
- **Defer permission asks until the feature needs them** — the permission-priming pattern:
  1. User initiates the feature that needs the permission.
  2. App shows a one-line in-UI explanation of why (the "primer") with the feature's benefit.
  3. Only then fire the system permission dialog.
  4. On denial: degrade gracefully, keep the feature entry visible with a path to settings —
     never re-prompt in a loop (Android stops showing the dialog after repeated denials anyway).
- Never ask for all permissions at launch: each unexplained launch-time prompt measurably raises
  uninstall/denial rates and poisons later asks.
- Account creation is never a prerequisite for local-only utility value.

## 8. Ad-UX Coexistence Rules

For ad-funded apps (deep doctrine in `08-knowledge/monetization/ads-patterns.md`; policy
boundaries in `08-knowledge/stores/policy-landmines.md` — re-verify before each release):

- **Never between intent and result.** No interstitial between "user taps Recover" and the
  recovery result. Acceptable interstitial moments: *after* a completed task, at natural session
  boundaries, on explicit exits from a flow.
- **Fixed positions.** Banners/natives own reserved layout slots that exist even while the ad
  loads — content never reflows when an ad arrives.
- **No layout-shift bait**: an ad must never appear where a button was a frame ago, or move
  content during scroll/tap. Accidental-click-engineered UIs are both a UX failure and a policy
  violation that gets accounts banned.
- Ads are visually distinguishable from content; close affordances are honest (real hit targets,
  no 1px X).

## 9. Trust Signals

Utility apps touch users' files and data; trust is the conversion currency.

- **Privacy moments**: at sensitive junctures (first storage access, first upload), one quiet line
  of reassurance — "Processing happens on your device" / "Nothing is uploaded". Matches the data
  safety form (`08-knowledge/stores/policy-landmines.md`) — claims must be true.
- **Transparent progress**: show *what* is being done ("Scanning DCIM… 1,204 photos found"), not
  just that something is happening. Specificity reads as competence.
- **Honest errors with next steps** (§3): blame the situation, not the user; never fake success.
- Visual quality itself is a trust signal: a dated UI (M2-era look — see
  `08-knowledge/design/material3-essentials.md`) lowers users' trust in the app's *safety*, not
  just its polish.

---

## Anti-Pattern Gallery

| Anti-pattern | What it looks like | Why it fails | Replace with |
|---|---|---|---|
| Mystery-meat navigation | Unlabeled icon-only controls for non-universal concepts | Users won't tap what they can't predict; features go undiscovered | Labels on nav bar items; text+icon for anything non-universal (search/back/share are the few safe icons) |
| Dialog stacking | Dialog opens dialog opens permission prompt | Disorienting; back behavior undefined; predictive back exposes the mess | One modal at a time; sequence flows as screens or sheets |
| Notification spam | "We miss you", daily tips, promo via the channel users allowed for downloads | Channel-level mute or app uninstall; POST_NOTIFICATIONS denial on next install | One channel per real purpose; promotional content only in opt-in channels |
| Fake urgency | Countdown timers on discounts that reset, "3 scans left today" artificial walls | Erodes trust permanently; review-bomb material; policy risk | Honest pricing and limits; value-based upsell at moments of demonstrated value |
| Settings as junk drawer | 40 toggles, ad consent buried, duplicated entries | Maintenance and support burden; signals indecision | Opinionated defaults; settings audit each redesign — every toggle must justify itself |
| Splash-screen theater | 3-second branded splash + loading screen on a local-only app | Delays the job (§1) for vanity | System SplashScreen API only, dismissed the instant the first frame is ready |
| Rating-prompt ambush | In-app review dialog on first launch or mid-task | Low ratings, prompt fatigue, policy limits on frequency | Prompt only after a *successful* core-task completion, capped, via the official in-app review API |
| Pull-to-refresh cargo cult | Refresh gesture on local-only screens with nothing to refresh | Teaches users a gesture that does nothing; masks real staleness signals | Refresh only where remote/changing data exists; local screens update reactively |
| Infinite confirmation toasts | Toast/snackbar for every trivial action ("Sorted by name") | Noise trains users to ignore feedback that matters | Confirm only non-obvious or destructive-adjacent outcomes; visible state change is its own confirmation |

---

Use in audits: score each principle section Pass / Partial / Fail with evidence (screenshot +
screen name) in `docs/factory/audits/DESIGN_AUDIT.md`; every Fail maps to a roadmap item with
severity per the scales in `01-core/CONVENTIONS.md`.
