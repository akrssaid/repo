# Workflow — Screenshot Optimization

This workflow turns store screenshots from a feature tour into a conversion asset: it defines the
message narrative, caption rules, production specs, localization handling, and the Play A/B
methodology for proving improvements. It is consumed by `01-core/prompts/11-screenshot-generation.md`
(production) and `01-core/prompts/14-store-assets.md` (listing assembly), and depends on the
differentiation thesis from `04-aso/workflows/competitor-analysis.md`. Conversion theory backing
this workflow lives in `08-knowledge/aso/conversion-optimization.md`.

## When to Run

| Trigger | Scope |
|---|---|
| First store-asset cycle for a target app (P7) | Full workflow |
| After a redesign lands (`01-core/prompts/09-ui-implementation.md`) — screenshots must show the current UI | Stages 2–5 |
| Listing conversion rate below category norms in Play Console | Stages 1–3 redesign + Stage 6 A/B |
| Differentiation thesis changed after a competitor refresh | Stage 2 re-sequencing + Stage 6 test |
| New locale added (`04-aso/workflows/localization.md`) | Stage 5 only |

## Required Inputs

- Differentiation thesis and positioning map from `docs/factory/reports/COMPETITOR_ANALYSIS.md`.
- Keyword clusters from `docs/factory/reports/KEYWORD_MAP.md` (captions should echo primary
  cluster language where natural — captions are conversion copy first, keyword carriers second).
- Current app UI (post-redesign build) and capture pipeline from `03-design/03-screenshot-capture.md`.
- Per-store asset specs from `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
  `04-aso/stores/rustore.md`.
- Play Console access (acquisition reports + store listing experiments).

## Procedure

### Stage 1 — Conversion Theory Recap (read before designing)

1. Internalize the decision model from `08-knowledge/aso/conversion-optimization.md`:
   - A store visitor decides in roughly **3 seconds** on the above-the-fold trio:
     **icon + rating + first screenshots**. Most visitors never scroll, never expand the
     description, never watch a video.
   - Therefore the **first 2–3 screenshots do nearly all the conversion work**. Screenshots 4–8
     serve the minority who scroll (deeper evaluators) — they matter, but never at the expense
     of the first three.
   - Screenshots are viewed first as **thumbnails in search results** (Play shows them inline for
     many queries). If a caption is unreadable at thumbnail size, it does not exist.

### Stage 2 — Narrative Design

2. **One benefit per screenshot.** Write the message set before any visual work: list candidate
   benefit messages from the differentiation thesis and the Exploit list in
   `docs/factory/reports/COMPETITOR_ANALYSIS.md`. Each screenshot carries exactly one message —
   a screenshot trying to say two things says nothing at thumbnail size.
3. **Sequence as a value story, not a feature tour.** Order by user motivation, not by app
   navigation:
   1. Screenshot 1 = the differentiation thesis claim (the reason to choose *this* app).
   2. Screenshot 2 = the core job being done (proof the app delivers the category promise).
   3. Screenshot 3 = the strongest objection-killer (e.g. "works offline", "no sign-up") —
      sourced from competitor 1-star themes.
   4. Screenshots 4–8 = secondary benefits in descending priority, ending with trust signals
      (privacy, no ads claim if true) where relevant.
   Never open with a settings screen, an empty state, or a splash screen.
4. **Caption rules** (apply to every screenshot):
   - **≤6 words.** Count them. Cut articles before cutting verbs.
   - **Benefit-led and verb-first**: "Recover photos in one tap", not "Photo recovery feature".
   - **Readable at thumbnail**: minimum ~1/15 of image height for caption text; test by viewing
     the asset at 25% zoom — if you squint, rewrite or enlarge.
   - No claims the app cannot deliver (policy and refund/uninstall risk —
     `08-knowledge/stores/policy-landmines.md`).

### Stage 3 — Production Specs

5. **Frame style consistency.** Define once, apply to all: caption position (top is standard —
   thumbnails crop bottoms less predictably), font (brand font, single weight pair), caption
   color, padding. All screenshots in a set share the same layout grid.
6. **Brand background.** Use a consistent brand-colored or brand-gradient background behind
   device content; it makes the set recognizable in a results list and visually distinct from
   competitors (check the positioning map — if every competitor uses blue, do not use blue).
7. **Device frame vs full-bleed decision rule:**
   - **Device frame** when the UI itself is the proof (redesigned, modern, attractive UI) and
     captions need room — frame shrinks UI but adds polish and caption space.
   - **Full-bleed** when in-app content is the product (reader content, media, files found) and
     detail must survive thumbnail scaling.
   - Pick one mode per set; mixing is acceptable only for a deliberate pattern-break on a
     trust-signal final screenshot.
8. **Text safe zones.** Keep captions inside the central 90% horizontally and out of the top/bottom
   5% (store UI overlays and rounded-corner masking). Nothing essential within 64 px of any edge
   at production resolution.
9. **Per-store dimensions and counts.** Produce per the specs in `04-aso/stores/google-play.md`,
   `04-aso/stores/huawei-appgallery.md`, and `04-aso/stores/rustore.md` (those files are
   authoritative for pixel sizes, aspect ratios, min/max counts, and feature-graphic
   requirements). Export from one master layout per screenshot; never upscale.
10. Hand production to `01-core/prompts/11-screenshot-generation.md` with: message list, sequence,
    captions, frame-style spec, and per-store export matrix. Capture raw UI per
    `03-design/03-screenshot-capture.md` on a clean device profile (realistic demo data, full
    battery/signal status bar or status bar cleaned, no debug overlays).

### Stage 4 — Localization of Captions

11. For each target locale from `04-aso/workflows/localization.md`: translate **meaning, not
    words** — captions are marketing copy; re-derive phrasing per locale and re-check the ≤6-word
    rule (German and Russian expand ~30%; if a translated caption exceeds the space, rewrite
    shorter rather than shrinking the font below thumbnail readability).
12. Re-export per-locale sets; UI content in the screenshot should show the localized app build
    for Tier 2+ locales (see scope tiers in `04-aso/workflows/localization.md`).

### Stage 5 — Measurement Baseline

13. Before changing anything live, record the baseline from Play Console acquisition reports:
    store-listing conversion rate (store listing visitors → installers), split by search vs
    browse traffic, last 28 days. Note Play's category-peer benchmark if shown. Store the
    baseline in `docs/factory/audits/ASO_AUDIT.md` (instance of `09-templates/aso-audit-report.md`;
    this is the only ASO_AUDIT path — see `01-core/CONVENTIONS.md`). AppGallery and RuStore expose
    less granular analytics — record what their consoles provide and label gaps.

### Stage 6 — Experiment Handoff (to the experiment program of record)

The experiment program — backlog, prioritization, sequencing rules, runtime/significance
thresholds, the canonical log, and the store-by-store A/B mechanics — is owned by
`04-aso/workflows/experiment-program.md`. This workflow does **not** restate those rules or run an
ad-hoc test; it produces a well-formed screenshot hypothesis and hands it to that program.

14. **One variable at a time.** A valid screenshot experiment changes exactly one thing:
    screenshot 1's message, OR the sequence order, OR caption style. Changing the whole set tells
    you nothing about why the result moved.
15. **What to test first: screenshot 1's message.** It has the largest exposure and the largest
    plausible effect (icon is tested before screenshots — see the program's "what to test first"
    ordering). Frame the hypothesis as thesis-claim vs strongest objection-killer as the lead
    message.
16. **Queue an EXP entry, do not log here.** Add the hypothesis to the backlog and let
    `04-aso/workflows/experiment-program.md` assign an `EXP-NNN` id and record the run in
    `docs/factory/reports/EXPERIMENT_LOG.md` (instance of `09-templates/experiment-log.md`). Provide:
    variable under test, the two variants, the audit/thesis finding that motivates it, and the
    target locale. Play store-listing-experiments mechanics (up to 3 variants vs the current
    listing) and AppGallery/RuStore sequential before/after method are documented there, not here.
    The conversion baseline from Stage 5 is the experiment's before-measurement reference.

## Outputs

| Artifact | Path |
|---|---|
| Screenshot narrative spec (messages, sequence, captions, frame style) | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` per `05-handover/ASO_HANDOVER_SCHEMA.md` |
| Final per-store, per-locale screenshot sets | `docs/factory/assets/store/<store>/<locale>/` |
| Conversion baseline | `docs/factory/audits/ASO_AUDIT.md` |
| Experiment entries (EXP IDs, results) | `docs/factory/reports/EXPERIMENT_LOG.md` via `04-aso/workflows/experiment-program.md` |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` |

## Acceptance Criteria

- [ ] Every screenshot carries exactly one benefit message; sequence follows
      thesis → core job → objection-killer → secondary.
- [ ] Screenshot 1 states the differentiation thesis claim from
      `docs/factory/reports/COMPETITOR_ANALYSIS.md`.
- [ ] All captions ≤6 words, benefit-led, verb-first, readable at 25% zoom.
- [ ] One frame style across the set; device-frame/full-bleed choice justified by the decision
      rule; safe zones respected.
- [ ] Per-store dimensions/counts match `04-aso/stores/*.md`; no upscaled exports.
- [ ] Localized caption sets re-written (not word-translated) for every target locale and
      re-checked for length and readability.
- [ ] Conversion baseline recorded in `docs/factory/audits/ASO_AUDIT.md` before any live change;
      first experiment queued as a single-variable hypothesis to
      `04-aso/workflows/experiment-program.md` (logged there with an EXP id, not in ASO_AUDIT); no
      screenshot shows UI or claims the shipped app does not deliver.
