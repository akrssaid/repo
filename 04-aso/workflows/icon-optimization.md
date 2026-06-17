# Workflow — Icon Optimization

This workflow treats the app icon as an ASO conversion lever: it audits the current icon's
performance readiness, defines single-variable iteration briefs, hands production to the design
track, and validates changes through Play icon experiments. The factory boundary is strict —
**ASO defines the goal and the test plan; design executes the craft** via
`03-design/06-icon-redesign.md`. This workflow is consumed by `01-core/prompts/12-aso-audit.md`
(audit), `01-core/prompts/10-icon-redesign.md` (production trigger), and
`01-core/prompts/14-store-assets.md` (per-store delivery).

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass (P7) — icon audit is part of `01-core/prompts/12-aso-audit.md` | Stages 1–2 |
| Search impressions are healthy but search CTR/conversion lags category norms | Full workflow |
| Brand redesign or visual overhaul landed (P6) | Stages 3–5 (brief, production, test) |
| Competitor refresh shows icon-shelf convergence (everyone now looks like you) | Stages 2–5 |
| New store launch (AppGallery/RuStore) | Stage 6 compliance pass only |

## Required Inputs

- Competitor capture sheets (icon strategy fields) from
  `docs/factory/reports/COMPETITOR_ANALYSIS.md` (`04-aso/workflows/competitor-analysis.md`).
- Current icon source assets and brand constraints from the design track
  (`03-design/04-design-system-extraction.md` output if available).
- Icon design principles: `08-knowledge/design/app-icon-principles.md`.
- Play Console access for store listing experiments and acquisition reports.
- Per-store icon specs in `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
  `04-aso/stores/rustore.md`.

## Procedure

### Stage 1 — Understand the Icon's ASO Role

1. Frame the work correctly before auditing:
   - The icon is the **most visible asset in search results** — it appears at small size next to
     every ranking the app earns, in search, charts, similar-apps rails, and the home screen
     after install. Screenshots appear on the listing page; the icon appears *everywhere*.
   - Its primary ASO function is **CTR**: tap-through from a results list into the listing
     (and, on Play, often direct install from search without a listing visit — for that path the
     icon plus rating is nearly the whole pitch).
   - A claim-free asset: the icon cannot explain — it can only be *seen, recognized, and chosen*.
     Optimize for legibility and differentiation, never for information density.

### Stage 2 — Audit Current Icon CTR-Readiness

2. **Legibility at 48 px.** Render the current icon at 48×48 px (the practical floor across
   store result lists and dense launchers). Pass criteria: the core symbol is identifiable, no
   element dissolves into noise, no text is present that becomes unreadable (icon text almost
   always fails at this size — flag any).
3. **Contrast vs Play surfaces.** Place the 48 px render on white `#FFFFFF` (Play light theme)
   and dark `#131314` (Play dark theme) swatches. Pass criteria: the icon holds a defined edge on
   both — near-white icons without an internal background fail light mode; near-black ones fail
   dark mode. Remember Play masks icons to rounded shapes and forbids relying on transparency
   beyond the keyline; verify against `04-aso/stores/google-play.md` specs.
4. **Category-shelf differentiation test.** Build a contact sheet: the current icon placed among
   the 10 competitor icons captured in `docs/factory/reports/COMPETITOR_ANALYSIS.md`, all at
   48 px, in a search-results-style column. Evaluate honestly:
   - Can a stranger pick out the app in under 2 seconds?
   - Does it share dominant color + symbol with 3+ competitors (shelf camouflage)?
   - Is it distinct *without* looking like it belongs to a different category (a file manager
     icon that reads as a game loses qualified taps)?
5. Score the audit (Pass / Marginal / Fail per test) into `docs/factory/audits/ASO_AUDIT.md` (the
   only ASO_AUDIT path — see `01-core/CONVENTIONS.md`).
   Any Fail, or two Marginals, justifies an iteration cycle. Record current search CTR proxy
   from Play Console (search impressions → listing visitors + search direct installs, 28 days)
   as the baseline.

### Stage 3 — Iteration Brief Rules

6. **One concept variable per test.** Each brief changes exactly one of:
   - **Color** — dominant hue/background change (e.g. blue → orange to break shelf camouflage),
     same symbol, same style.
   - **Symbol** — different core mark (e.g. magnifier → restored-photo glyph), same palette,
     same style.
   - **Style** — same symbol and palette, different rendering (flat → gradient depth, outline →
     filled, with/without container shape).
   A brief that changes two of these produces an untestable result — if it wins you cannot tell
   why, and the next iteration has no foundation.
7. Each brief must state: the variable under test, the hypothesis ("orange background will lift
   search CTR because 7/10 shelf competitors are blue — see contact sheet"), the audit failure it
   addresses, what must NOT change, and the success metric (Play experiment result).
8. Sequence briefs by audit severity: fix legibility/contrast Fails first (they cap everything),
   then differentiation, then style polish.

### Stage 4 — Production Handoff (factory boundary)

9. Hand the brief to the design track via `03-design/06-icon-redesign.md` (executed through
   `01-core/prompts/10-icon-redesign.md`). **ASO defines the goal and the test plan; design owns
   craft decisions** — grid construction, optical balance, adaptive-icon layering, asset export.
   ASO does not art-direct strokes and corners; design does not silently change a second variable.
10. Acceptance gate on returned candidates: re-run Stage 2 tests (48 px legibility, dual-surface
    contrast, contact-sheet differentiation) on each candidate before any goes live. A candidate
    failing the audit it was briefed to fix goes back with the contact-sheet evidence.

### Stage 5 — A/B Testing on Play (icon experiments)

The experiment program of record — sequencing, runtime/significance thresholds, the canonical log,
and per-store mechanics — is owned by `04-aso/workflows/experiment-program.md`. **Icon is the first
asset that program tests** (largest, most universal surface; screenshot 1 follows). This stage
supplies the icon-specific discipline that program enforces.

11. Configure a Play Console **store listing experiment** with the icon as the only changed
    element. Use the main store listing (or highest-traffic custom listing); 1 variant at 50/50
    split is the default — use up to 3 variants vs the current listing only if the listing has high
    traffic (icon experiments need substantial impressions because per-impression effect sizes are
    small). Mechanics and the 3-variant cap are detailed in `04-aso/workflows/experiment-program.md`.
12. **Single-variable discipline (hard rule, keep).** One concept variable per experiment (the
    color, symbol, OR style isolated in Stage 3); one icon experiment at a time; never run
    concurrently with a screenshot experiment on the same listing
    (see `04-aso/workflows/screenshot-optimization.md`). The sequencing/settle rules live in the
    program file.
13. **Decision and the post-apply retained-installer check (keep):**
    - **Apply** the variant if Play reports a confident positive result on installs.
    - **Keep control** if confidently negative or inconclusive — an inconclusive icon test usually
      means the variable was too timid; brief a bolder variant.
    - Watch the **post-apply window** (14 days): confirm no decline in retained installers (Play
      reports retained-installer conversion). An icon that buys taps from the wrong users shows
      up here — roll back if retained-install conversion drops.
14. **Log via the program, with an EXP id.** Record every experiment (hypothesis, variable, variant,
    dates, result, decision, next action) in `docs/factory/reports/EXPERIMENT_LOG.md` (instance of
    `09-templates/experiment-log.md`) through `04-aso/workflows/experiment-program.md` — not in
    ASO_AUDIT. The icon CTR-readiness audit and its scores stay in `docs/factory/audits/ASO_AUDIT.md`.

### Stage 6 — Multi-Store Consistency Policy

14. **Same core mark everywhere.** The winning symbol, palette, and style ship to Play,
    AppGallery, and RuStore — a user who saw the app on one store must recognize it on another,
    and brand assets (website, ads) must match.
15. **Store-specific compliance only.** Per-store exports may differ in technical envelope, never
    in identity: dimensions, corner/masking rules, padding/keyline, and badge/decoration policies
    per `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
    `04-aso/stores/rustore.md`. AppGallery and RuStore have no icon A/B tooling — Play-validated
    winners roll out there, monitored via per-store conversion deltas recorded manually.
16. Update the in-app launcher icon (adaptive icon resources) in the same release that changes
    the store icon — a store/launcher mismatch breaks the recognition chain at the highest-intent
    moment (post-install first launch). Coordinate via `01-core/prompts/10-icon-redesign.md`.

## Outputs

| Artifact | Path |
|---|---|
| Icon CTR-readiness audit (scores, contact sheet findings, CTR baseline) | `docs/factory/audits/ASO_AUDIT.md` |
| Iteration brief(s) | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` per `05-handover/ASO_HANDOVER_SCHEMA.md` |
| Final per-store icon assets | `docs/factory/assets/store/<store>/` |
| Experiment entries (EXP IDs, results) | `docs/factory/reports/EXPERIMENT_LOG.md` via `04-aso/workflows/experiment-program.md` |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` |

## Acceptance Criteria

- [ ] Audit completed with explicit Pass/Marginal/Fail per test (48 px legibility, light/dark
      contrast, shelf differentiation) and a dated contact sheet against the current competitor set.
- [ ] Search CTR baseline recorded from Play Console before any live change.
- [ ] Every iteration brief isolates exactly one variable (color, symbol, or style) and states a
      hypothesis tied to an audit finding.
- [ ] Production executed via `03-design/06-icon-redesign.md`; candidates re-pass Stage 2 tests
      before going live.
- [ ] Icon experiment logged with an EXP id in `docs/factory/reports/EXPERIMENT_LOG.md` via
      `04-aso/workflows/experiment-program.md`; decision made only on Play-reported significance;
      post-apply retained-installer check done; no concurrent screenshot experiment.
- [ ] Same core mark across all stores; per-store exports comply with `04-aso/stores/*.md`;
      launcher icon updated in the same release.
