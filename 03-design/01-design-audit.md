# Design Audit Method

This document defines the full UX/design audit method executed in P5 by
`01-core/prompts/05-design-audit.md`. It specifies the heuristics framework, the per-screen scoring
rubric, evidence requirements, finding classification and severity rules, and the DES-xxx finding
format. The audit's output is a populated instance of `09-templates/design-audit-report.md` saved
to `docs/factory/audits/DESIGN_AUDIT.md` in the target repo (path per `01-core/CONVENTIONS.md`).

**The H1–H9 per-screen rubric defined below is THE design-audit rubric** — the one cited by the
`09-templates/design-audit-report.md` per-screen scorecard and by `01-core/prompts/05-design-audit.md`.
The template's old 10-heuristic per-app table is abolished; mobile-UX principles appear only as a
separate Pass/Partial/Fail summary, never as a second scorecard (see `01-core/CONVENTIONS.md` §Design
contracts). All IDs, scales, and paths in this doc are the canonical ones from `01-core/CONVENTIONS.md`.

## Preconditions

| Input | Source | Hard requirement |
|---|---|---|
| Screen inventory with SCR-IDs | `03-design/02-screen-inventory.md` → `docs/factory/audits/SCREEN_INVENTORY.md` | Yes — audit scope is exactly the inventory |
| Baseline screenshots (all states) | `03-design/03-screenshot-capture.md` → `docs/factory/assets/screenshots/baseline/` | Yes — no screenshot, no finding |
| Current design system extraction | `03-design/04-design-system-extraction.md` | Yes — needed for consistency scoring |
| App category playbook | `06-playbooks/<category>.md` if one exists | No — use if present for category norms |

If any hard requirement is missing, stop and execute the missing method first. Do not audit from
partial evidence.

## The Nine Heuristics

Score every screen on all nine. Definitions are operational — each lists what to actually check.

| # | Heuristic | What to check on each screen |
|---|---|---|
| H1 | Visual hierarchy | Is the primary action visually dominant? Is there exactly one focal point? Do size/weight/color rank elements by importance? Is text scannable (clear heading vs body distinction)? |
| H2 | Consistency | Same components for same jobs across screens; colors/type/spacing match the extracted design system; icon style uniform; terminology identical for identical concepts (e.g., "Delete" never becomes "Remove"). |
| H3 | Feedback | Every tap gives a visible response < 100 ms (ripple, state change); destructive actions confirm; long operations show progress; success/failure of user actions is announced (snackbar, state change). |
| H4 | Navigation clarity | User always knows where they are (titles, selected nav item); back behaves predictably (system back = up unless documented); reachable in ≤ 3 taps from home for core tasks; no dead ends. |
| H5 | Error handling | Errors say what happened, why, and what to do next, in human language; retry offered where retry can work; input validation is inline and before submit; no raw exception text or error codes alone. |
| H6 | Empty states | First-run/zero-data states explain what belongs here and offer the action that fills it; no blank white screens; no spinner-forever when the answer is "empty". |
| H7 | Loading states | Skeletons or progress indicators for loads > 300 ms; content does not jump when loaded (reserved space); cached content shown stale-while-revalidate where applicable; no UI freeze. |
| H8 | Touch ergonomics | All targets ≥ 48×48 dp; primary actions in thumb zone (bottom half) on traffic-H screens; destructive actions not adjacent to frequent actions; gestures have visible alternatives. |
| H9 | Platform conventions | Material component vocabulary (not iOS patterns); system back/predictive back works; edge-to-edge handled (no content under system bars without insets); dark theme honored; state preserved across rotation and process death. |

**Motion & transition quality (audit home).** Motion is not a tenth heuristic, but motion findings
need an audit home. Assess transition/animation quality under **H9** (platform conventions: does
predictive-back animate, do navigation transitions match the relationship, is there jank?) and
**H3** (feedback: is the response to a tap animated and timely?). When a screen's transitions are
absent, janky, missing a reduced-motion fallback, or mismatched to the navigation relationship,
record it as a DES finding tagged with H9 (or H3 for feedback-level motion) and note `motion` in the
finding title; the recommendation feeds `03-design/10-motion-design.md`. The motion *target* spec is
authored in the redesign, not the audit — the audit only documents the current motion gap (raw
material comes from the Pass 8 motion census in `03-design/04-design-system-extraction.md`).

## Scoring Rubric (1–5, per heuristic, per screen)

| Score | Meaning | Operational definition |
|---|---|---|
| 5 | Exemplary | Meets every check for the heuristic; could be used as a reference example |
| 4 | Good | Minor cosmetic deviations only; no user-visible friction |
| 3 | Adequate | Works, but at least one check fails in a way users will notice |
| 2 | Poor | Multiple checks fail; users hesitate, mis-tap, or get confused |
| 1 | Broken | The heuristic's purpose is defeated; users get stuck, lost, or harmed |

Scoring procedure:

1. Open the screen's baseline screenshots (all captured states: default, empty, loading, error,
   offline, dark — the states contract per `03-design/02-screen-inventory.md`) side by side.
2. Walk the heuristic's check column; record each failed check as a bullet with the screenshot
   path that shows it.
3. Assign the score from the rubric. A score of 1 or 2 on any heuristic **must** produce at least
   one DES finding; a 3 should usually produce one.
4. Record scores in the per-screen scorecard table (format below). Any heuristic that does not
   apply to a screen (e.g., H6 on a settings screen with no data collection) is marked `n/a` and
   excluded from averages — never scored 5 by default.

Per-screen scorecard row format (one row per screen in the report):

| SCR-ID | H1 | H2 | H3 | H4 | H5 | H6 | H7 | H8 | H9 | Avg | Priority |
|---|---|---|---|---|---|---|---|---|---|---|---|
| SCR-001 | 3 | 2 | 4 | 4 | 2 | n/a | 3 | 4 | 2 | 3.0 | H |

`Priority` is the traffic priority (H/M/L) copied from the screen inventory — it weights which
screens' low scores matter most for the redesign scope.

## Evidence Requirements

Every finding and every score ≤ 3 must cite evidence:

1. **Screenshot path** — repo-relative path under `docs/factory/assets/screenshots/baseline/`,
   following the `SCR-<id>-<state>-<theme>.png` convention from `03-design/03-screenshot-capture.md`.
2. **Annotation** — either (a) an annotated copy saved next to the original with suffix
   `-annotated` (e.g., `SCR-004-error-light-annotated.png`) with a rectangle/arrow on the problem
   region, or (b) a textual locator in the finding: region of screen + element label
   (e.g., "bottom-right FAB", "third list row, trailing icon").
3. **Reproduction note** for state-dependent findings — one line on how the state was reached
   (e.g., "airplane mode on, pull-to-refresh").

Findings without evidence are deleted at review, not argued about. If a state could not be
captured (e.g., error state unreachable), say so explicitly in the report's Coverage section with
the reason — silence is treated as a coverage gap.

## Finding Classification: Systemic vs Per-Screen

Classify every finding as one of:

- **Systemic (SYS)** — root cause is in the design system or a shared component, and the symptom
  appears on ≥ 3 screens or on every use of a component. Examples: hardcoded hex colors bypassing
  the theme; inconsistent button heights from per-screen styling; no app-wide loading pattern.
  Record once, list all affected SCR-IDs, fix once at the token/component level.
- **Per-screen (SCR)** — root cause is local to one screen's layout or flow. Examples: a paywall
  with the dismiss target overlapping the purchase button; one screen ignoring dark theme.

Classification rule of thumb: if the fix lives in `themes.xml`/`Theme.kt`/a shared composable or
style, it is systemic. If the fix lives in one screen's layout/composable, it is per-screen.
Systemic findings are fixed before per-screen findings in the redesign (doctrine: tokens before
screens — see `03-design/README.md`).

## Severity Mapping

Use the canonical scale (Critical / High / Medium / Low). Map by user and business impact, not by
how ugly it is:

| Severity | Assign when… | Examples |
|---|---|---|
| Critical | Blocks a core task; a monetization surface is broken or deceptive; data-destructive action without confirmation; legally risky (consent dialog unreadable); app unusable at font scale 1.3 or in dark mode (crash/invisible text) | Paywall purchase button off-screen on 1080×2400; delete-all with no confirm; white-on-white text in dark theme on main screen |
| High | Core task significantly degraded; heuristic scored 1 on a traffic-H screen; accessibility blocker (touch target < 32 dp on primary action, contrast < 3:1 on body text) | No error state on main feed (infinite spinner offline); primary CTA below the fold |
| Medium | Noticeable friction on H/M screens; heuristic scored 2; inconsistency users will perceive | Mixed icon styles in bottom nav; three different button corner radii |
| Low | Cosmetic; only on traffic-L screens; polish items | 2 dp misalignment; outdated divider style on licenses screen |

Escalation rules: any finding on a monetization surface starts one severity higher than the table
suggests (minimum High). Any finding that also fails a gate in `07-checklists/accessibility.md`
is at minimum High.

## Finding Format: DES-xxx

IDs are `DES-` + zero-padded 3-digit sequence (`DES-001`, `DES-002`, …), unique for the lifetime of
the project — never renumber, never reuse retired IDs. Every finding uses exactly this block:

```markdown
### DES-014 — Error state missing on document list (SCR-003)
- **Severity:** High
- **Class:** SCR  (or SYS + affected: SCR-003, SCR-007, SCR-011)
- **Heuristic:** H5 — Error handling
- **Screens:** SCR-003
- **Evidence:** docs/factory/assets/screenshots/baseline/SCR-003-error-light.png
  (annotated: ...-annotated.png) — spinner persists indefinitely with network disabled
- **Repro:** enable airplane mode, open Documents tab
- **Impact:** user cannot distinguish "loading" from "failed"; abandons core task
- **Recommendation:** offline error state with retry; pattern to be defined once (SYS candidate
  if same gap exists on other list screens)
- **Effort:** S
```

`Effort` uses the canonical S/M/L/XL scale. `Recommendation` describes direction, not pixel-level
specs — pixel specs belong to the redesign (`01-core/prompts/07-design-redesign.md`).

### Accessibility findings use the A11Y-xxx stream (not DES)

Accessibility findings have their **own ID stream**: `A11Y-001`, `A11Y-002`, … (format per
`01-core/CONVENTIONS.md` §ID registry). The old `A11Y-DES-xxx` format is abolished — never use it.
DES findings are design/UX findings; A11Y findings are accessibility-gate findings (contrast, touch
target, content description, font-scale survival, TalkBack order). When a problem is both a design
defect and an accessibility-gate failure, raise the DES finding for the design fix and a separate
`A11Y-NNN` finding for the gate, and cross-reference them by ID. The A11Y findings are authored by
`03-design/09-accessibility-review.md` and consume the same evidence/severity rules as this doc.
A DES finding that also fails a `07-checklists/accessibility.md` gate is still at minimum High (see
escalation rule above), and must name the paired `A11Y-NNN` finding.

## Audit Procedure

1. Load the screen inventory; confirm every SCR-ID has baseline screenshots for all states marked
   required in the inventory. List gaps; capture missing states before scoring
   (`03-design/03-screenshot-capture.md`).
2. Score all screens against H1–H9 using the rubric; build the scorecard table. Audit in traffic
   priority order (H screens first) so time pressure cannot silently drop the screens that matter.
3. Write DES findings as you score — one finding per distinct problem, not per symptom sighting.
   When the same symptom recurs, add the SCR-ID to the existing finding and re-evaluate SYS vs SCR.
4. After all screens are scored, do a cross-screen consistency pass (H2 at app level): compare
   screenshots of all H-priority screens in a grid; log systemic inconsistencies that no single
   screen reveals.
5. Audit monetization surfaces against `08-knowledge/monetization/ads-patterns.md` and
   `08-knowledge/monetization/subscription-patterns.md` (placement legality, accidental-click
   patterns, paywall clarity). Apply the severity escalation rule.
6. Apply severity and effort to every finding; sort the findings register by severity, then by
   traffic priority of affected screens.
7. Populate `09-templates/design-audit-report.md` and save as
   `docs/factory/audits/DESIGN_AUDIT.md`.

## Feeding the Report Template

Map audit outputs into `09-templates/design-audit-report.md` sections as follows:

| Audit output | Report template section |
|---|---|
| Scorecard table (all screens × H1–H9) | Heuristic Scorecard |
| DES findings, sorted | Findings Register |
| SYS findings summary + token divergences (from `03-design/04-design-system-extraction.md`) | Systemic Issues |
| Coverage gaps (uncapturable states, unscored screens + reasons) | Coverage & Limitations |
| Top-10 list: Critical + High findings on traffic-H screens | Executive Summary / Redesign Priorities |

## Quality Bar (self-check before declaring done)

- [ ] Every inventory SCR-ID appears in the scorecard (or in Coverage & Limitations with reason).
- [ ] Every score ≤ 3 traces to at least one evidence-backed observation.
- [ ] Every DES finding has all nine fields of the format, including effort.
- [ ] No finding mixes multiple root causes; SYS findings list all affected SCR-IDs.
- [ ] Monetization surfaces explicitly audited (or explicitly recorded as absent).
- [ ] Motion/transition quality assessed under H9/H3; any motion gap recorded as a DES finding
      whose recommendation feeds `03-design/10-motion-design.md`.
- [ ] Accessibility findings filed in the `A11Y-NNN` stream (never `A11Y-DES-xxx`); DES↔A11Y pairs
      cross-referenced by ID.
- [ ] Findings register count reconciles: every scorecard cell with 1–2 maps to ≥ 1 finding.
- [ ] Report saved to `docs/factory/audits/DESIGN_AUDIT.md`; `docs/factory/PROJECT_STATE.md`
      and `docs/factory/CHANGELOG.md` updated per `01-core/prompts/05-design-audit.md`.
