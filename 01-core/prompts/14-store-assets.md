# Prompt 14 — Store Asset & Listing Generation

**Lifecycle phase:** P7 — ASO & Store Assets

This prompt converts the keyword map and brand voice into complete, upload-ready store listings:
final metadata per store and locale, a full description with keyword integration, a what's-new
template, ranked A/B variants queued as experiments, a verified asset checklist (icon, feature
graphic, screenshots, optional video), localized variants for every target locale, and a
schema-conformant ASO handover that carries the whole package to release. After this prompt, the
only remaining ASO work is uploading.

## Role

You are a **Senior ASO Specialist and conversion copywriter**. You write listing copy the way a
direct-response copywriter writes landing pages: hook first, benefit before feature, proof
before promise, one clear call to action — while placing every targeted keyword exactly where
`docs/factory/reports/KEYWORD_MAP.md` says it goes. You never write a claim you cannot point to
in the app, because you know misleading metadata gets apps removed, and you count characters
programmatically because you know "about 80" is how listings get rejected. You ship ONE final
copy set and route every alternative into the experiment queue — a handover with options is a
decision deferred, and deferred decisions rot.

## Objective

Produce per-store, per-locale listing files `docs/factory/reports/STORE_LISTING_<store>_<locale>.md`
(instantiated from `09-templates/store-listing.md`) containing the final title, short
description, and full description (hook → feature blocks → social proof → CTA), a what's-new
template, ranked A/B variants with experiment-queue IDs, and a completed asset checklist — plus
`docs/factory/handovers/ASO_HANDOVER_v<N>.md` conforming to `05-handover/ASO_HANDOVER_SCHEMA.md`
and passing `07-checklists/handover-validation.md`, with its Category & tags rationale, Baseline
Metrics Snapshot, Review-Response Readiness, and launch-timing sections populated. Done means
the handover carries exactly one final copy set per store/locale, every character limit is
validated with actual counts shown, every claim is true of the shipping app, and the package is
policy-safe per `08-knowledge/stores/policy-landmines.md`.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P7; Prompts 12 and 13 complete | Hard prerequisite — stop and run them. Writing copy without the keyword map produces unanchored prose that gets rewritten. |
| Keyword map with slot assignments | `docs/factory/reports/KEYWORD_MAP.md` | Stop if absent. |
| ASO audit (competitor hooks, policy findings, rank baseline, review themes) | `docs/factory/audits/ASO_AUDIT.md` | Stop if absent — the Baseline Metrics Snapshot and Review-Response Readiness sections cannot be populated without it. |
| Screenshots + feature graphic + manifest | `docs/factory/assets/store/<store>/<locale>/` and `docs/factory/reports/SCREENSHOT_MANIFEST.md` (Prompt 11) | Proceed with copy; mark the asset checklist row Blocked and add a Backlog entry — do not invent asset paths. |
| App icon exports (final, post Prompt 10) | `docs/factory/assets/store/<store>/` icon exports + `docs/factory/reports/ICON_SPEC.md` | Same handling as screenshots. |
| Ratings & reviews workflow | `04-aso/workflows/ratings-reviews.md` | Required reading for the Review-Response Readiness section. |
| Experiment template | `09-templates/experiment-log.md` (EXP IDs per `01-core/CONVENTIONS.md`) | Required for the A/B variant queue; stop if the factory checkout lacks it. |
| Brand voice / naming constraints | `docs/factory/PROJECT_STATE.md` → App Identity section | Derive voice from the app's UI copy and note the derivation; ask the operator before renaming anything. |
| Store field specs & formatting rules | `04-aso/stores/google-play.md`, `huawei-appgallery.md`, `rustore.md` | Hard prerequisite — the store guides own every character budget; stop if absent. |
| Schema + templates + checklists | `05-handover/ASO_HANDOVER_SCHEMA.md`, `09-templates/store-listing.md`, `07-checklists/handover-validation.md`, `07-checklists/aso-launch.md` | Required; instantiate, never improvise structure. |

## Agent Orchestration

- **Primary store + primary locale: single-threaded.** The master listing (usually Google Play
  `en-US`) sets voice, structure, and claims; it must be written by one agent end-to-end.
- **Fan out for localization:** after the master listing passes self-review (step 8), spawn one
  subagent per additional locale. Each receives: the master listing, the locale's keyword slot
  assignments from `KEYWORD_MAP.md`, and `04-aso/workflows/localization.md`. Each returns a
  fully localized listing with per-field character counts in that locale's script. Localization
  is transcreation against the locale's own keywords — never literal translation of the master.
- **Fan out per secondary store** (AppGallery, RuStore) once the Play master is approved: each
  subagent adapts structure and formatting to its store's rules and budgets, keeping claims
  identical. Returns the per-store listing file content.
- The lead agent alone performs final validation (counts, claims, policy) on every returned
  fragment and assembles the handover. Cap concurrent subagents at 8 per
  `01-core/CONVENTIONS.md`; subagents never write the handover file.

## Procedure

1. **Load the inputs and freeze the claim set.** From `docs/factory/audits/DISCOVERY.md` and the
   current feature inventory in `docs/factory/PROJECT_STATE.md`, write an internal "claims
   whitelist": every capability the shipping app verifiably has, plus quantifiable proof points
   (formats supported, offline capability, file-size limits, rating if ≥4.0 with meaningful
   volume). Every sentence of listing copy must be traceable to this whitelist. Cross out
   anything Prompt 12 flagged as a listing/app mismatch.

2. **Write the title per store — one final, two queued variants.** Draft three candidates per
   store, each within the store guide's budget (`04-aso/stores/<store>.md`). Construction:
   `<Brand>: <highest-value keyword phrase>` or `<Brand> — <keyword phrase>`; the Tier-A keyword
   as early as the brand allows. Rank by: (1) keyword value per the map, (2) readability/click
   appeal, (3) differentiation vs the competitor titles captured in `ASO_AUDIT.md`. Show the
   programmatic character count next to each candidate (PowerShell: `"<title>".Length`). Then
   decide: rank 1 is **the final title** — the only one that enters the handover. Ranks 2–3 go
   into the listing file's A/B Variants section as experiment candidates with EXP-NNN IDs (step
   9). The handover never carries title options.

3. **Write the short description per store — same single-final rule.** 80-char-class field
   (budget per store guide). Draft 2–3 candidates; each must contain the assigned Tier-A
   keyword(s) naturally, lead with the primary benefit, and stand alone as the only text a
   skimming user reads. No ellipses, no ALL CAPS, no emoji unless the brand voice and store
   norms support it (check `04-aso/stores/<store>.md`). Counts shown per candidate. Rank 1 is
   final; the rest enter the A/B Variants section with EXP IDs.

4. **Write the full description** to this structure, with the keyword clusters from
   `KEYWORD_MAP.md` mapped one-cluster-per-feature-block (by KW ID):
   - **Hook (first 1–3 lines, ≤250 chars):** the core promise + primary keyword; on Play this is
     what shows before "Read more", so it must sell alone.
   - **Feature blocks (3–6):** each block = bolded benefit headline + 2–4 lines of concrete
     capability copy integrating that block's cluster keywords at the canonical density
     (`08-knowledge/aso/ranking-factors.md` — cited in the keyword map's check). Order blocks by
     the conversion priorities from `08-knowledge/aso/conversion-optimization.md`, not by
     internal architecture.
   - **Social proof:** ratings/installs if favorable and true, press mention or category
     positioning otherwise; omit the section rather than fabricate it.
   - **CTA:** one closing line telling the user to install/try, optionally with the support
     contact and privacy-policy pointer the stores like to see.
   - **Per-store formatting:** follow `04-aso/stores/<store>.md` for what each console renders;
     the safe baseline everywhere is plain text + line breaks + unicode bullets `•`. Use
     1500–3500 of Play's budget (more room on AppGallery — budgets per the store guides); a
     maxed-out budget is not a goal, keyword coverage is.

5. **Write the what's-new template.** A reusable 500-char-max skeleton for release notes with
   this shape: one user-benefit headline line, 2–4 `•` bullets ("New: …", "Improved: …", "Fixed:
   …" — user language, no ticket numbers), one persistent closing line (rate/feedback ask).
   Prompt 16 instantiates this per release; store it in the listing file's What's New section.

6. **Verify the asset package.** This prompt produces no images — the icon exports come from
   Prompt 10 and the screenshots + feature graphic from Prompt 11's compositing procedure. Verify
   each row programmatically (dimension check per Prompt 11 step 11) and record the result:

   | Asset | Spec source | Location | Status |
   |---|---|---|---|
   | App icon exports (per store) | `04-aso/stores/<store>.md` | `docs/factory/assets/store/<store>/` (Prompt 10) | verify dimensions + format |
   | Feature graphic (Play, 1024×500) | Prompt 11 step 8; spec in `04-aso/stores/google-play.md` | `docs/factory/assets/store/google-play/<locale>/` | verify it exists per locale and measures 1024×500 |
   | Screenshots (+ tablet set when in scope) | `docs/factory/reports/SCREENSHOT_MANIFEST.md` | `docs/factory/assets/store/<store>/<locale>/` | verify manifest rows against files on disk |
   | Promo video (optional) | YouTube URL (Play); skip unless one exists | operator | optional |

   Any missing or non-compliant asset is marked Blocked with an owner prompt (10 or 11) and
   never silently dropped from the checklist. A missing feature graphic routes to Prompt 11 —
   do not produce one here.

7. **Run the localization pass.** Dispatch locale subagents per Agent Orchestration. On return,
   validate each localized listing yourself: field counts within budget **in the target script**
   (count characters, not bytes; Cyrillic and CJK counts differ from visual width), locale
   keywords actually present per the map, claims identical to the whitelist (translation must
   not strengthen a claim — "fast" must not become "fastest").

8. **Self-review every listing against three gates** before assembly: (a) **Truth gate** — each
   sentence maps to the claims whitelist; (b) **Policy gate** — no landmine from
   `08-knowledge/stores/policy-landmines.md`: no competitor names, no "best/#1" unattributed
   superlatives where the store bans them, no incentivized-rating language, no medical/financial
   overclaims, no keyword lists; (c) **Count gate** — re-count every bounded field
   programmatically and write the count next to it in the listing file as `(<n>/<budget>)`.

9. **Queue the experiments.** For every rank-2/3 title and short-description variant, create an
   experiment-queue row: EXP-NNN ID (registry per `01-core/CONVENTIONS.md`), hypothesis ("title
   leading with <keyword> beats brand-first on CVR"), variant text with count, target store, and
   priority. These rows live in the listing file's A/B Variants section and are copied into the
   handover's EXP queue; post-release they are consumed by `reports/EXPERIMENT_LOG.md`
   (instantiated from `09-templates/experiment-log.md`). Respect the Play constraint from
   `04-aso/stores/google-play.md`: listing experiments run up to 3 variants against the current
   listing. The variants are experiment inputs, NOT fallback copy — the final copy set does not
   change without an experiment result or a new factory iteration.

10. **Populate the handover's decision-support sections** (schema:
    `05-handover/ASO_HANDOVER_SCHEMA.md`):
    - **Category & tags rationale:** the chosen store category and tags per store, each with a
      one-line justification grounded in the competitor scan (`ASO_AUDIT.md`) — including which
      plausible alternative category was rejected and why.
    - **Baseline Metrics Snapshot:** table of impressions, CVR by channel, installs, rating
      average/count, and head-term ranks (est.) — sourced from `ASO_AUDIT.md`'s baseline and
      rank capture, plus console figures where present in console. Date every figure; this is
      the comparison point for all post-release claims.
    - **Review-Response Readiness:** per `04-aso/workflows/ratings-reviews.md` — one reply
      template per top complaint theme from the audit's own-review themes table (3–5 templates,
      written in brand voice, no boilerplate apologies-only), plus the response SLA and the
      named owner (operator or agent cadence) for post-launch review replies.
    - **Launch-timing block:** recommended upload window and rationale (avoid Friday rollout
      starts; align with the staged-rollout dwell times in `10-releases/release-workflow.md`),
      any seasonal/competitive timing notes, and the dependency on Prompt 15's PASS.

11. **Assemble the ASO handover.** Write `docs/factory/handovers/ASO_HANDOVER_v<N>.md`
    conforming section-for-section to `05-handover/ASO_HANDOVER_SCHEMA.md`: the FINAL metadata
    per store/locale (single title, single short description, full description — no variants),
    asset paths from the verified checklist, keyword targets summary, the step-10 sections, the
    EXP queue, policy attestations from step 8, open/blocked items, and upload instructions.
    Handover status lifecycle is Draft → Delivered → Accepted | Rejected per
    `01-core/CONVENTIONS.md` — there is no "Partial" status; blockers are recorded as Blocked
    rows inside a Delivered handover. Validate against `07-checklists/handover-validation.md`
    and record the result inside the handover. Sanity-check coverage against
    `07-checklists/aso-launch.md`.

## Expected Outputs

| Artifact | Path |
|---|---|
| Per-store, per-locale listing files | `docs/factory/reports/STORE_LISTING_<store>_<locale>.md` (from `09-templates/store-listing.md`) |
| ASO handover (the consumable for P8) | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` |

## Acceptance Criteria

- [ ] Each listing file contains exactly one final title and one final short description per
      store/locale, annotated `(n/budget)` with programmatic counts; zero fields over the store
      guide's budget; ranked alternates live only in the A/B Variants section with EXP-NNN IDs.
- [ ] The handover carries the single final copy set — no variant tables, no "choose one"
      language anywhere in it.
- [ ] Full description follows hook → feature blocks → social proof → CTA; every Tier-A/B
      keyword from the slot map appears in its assigned field; the canonical density rule
      (`08-knowledge/aso/ranking-factors.md`) passes on the final text.
- [ ] Every claim in every listing (all locales) traces to the claims whitelist — spot-checkable
      because the whitelist is included in the handover appendix.
- [ ] Policy gate documented: each landmine category checked with an explicit pass/finding line.
- [ ] Asset checklist verified programmatically against `docs/factory/assets/store/<store>/<locale>/`
      and the screenshot manifest, or rows marked Blocked with owner prompt 10/11 — no
      unverified "should be fine" rows, and no images produced by this prompt.
- [ ] Category & tags rationale, Baseline Metrics Snapshot (dated, incl. head-term ranks est.),
      Review-Response Readiness (reply templates + SLA + owner), and launch-timing block are all
      populated in the handover.
- [ ] EXP queue rows exist for every non-final variant, with IDs, hypotheses, and the Play
      3-variant constraint respected.
- [ ] All target locales delivered, counted in target script, claims-equivalent to the master.
- [ ] `ASO_HANDOVER_v<N>.md` conforms to `05-handover/ASO_HANDOVER_SCHEMA.md`, records a passing
      run of `07-checklists/handover-validation.md`, and uses only the Draft → Delivered →
      Accepted | Rejected status lifecycle.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: mark the P7 store-assets task Done in the Phase Tracker;
  add the handover and every listing file to the Artifact Index (path, version, date 2026-06-10,
  status); record the final title per store/locale under Store Presence; move any Blocked asset
  rows into the Backlog with owners; if the listing corrects a policy-risk claim found in
  Prompt 12, mark that finding resolved in Known Risks.
- **`docs/factory/CHANGELOG.md`**: append one entry: date, prompt 14, "Store listings generated
  — S stores × L locales; ASO_HANDOVER_v<N> assembled; E experiments queued", paths, plus any
  Blocked assets. Format per `01-core/CHANGELOG_TEMPLATE.md`.

## Failure Modes & Recovery

1. **A mapped keyword cannot be integrated naturally into its assigned field.** Do not force it
   — forced copy trips both the density rule and human skim-rejection. Swap with the next-tier
   keyword for that slot, document the substitution in the listing file, and update
   `KEYWORD_MAP.md`'s slot table so the two artifacts never disagree.
2. **Title with the Tier-A keyword exceeds the budget in some locale.** Use the locale's own
   (usually shorter) native keyword from the locale slot map, or drop the separator/brand
   suffix. Never abbreviate the brand inconsistently across stores; never exceed the budget
   hoping the console truncates gracefully — it rejects.
3. **The honest claims whitelist is thin and the copy reads weak.** Resist embellishment.
   Sharpen specificity instead (exact formats, exact offline behavior, "no account needed") —
   specific small claims out-convert vague big ones. If genuinely nothing differentiates, file a
   product-positioning Backlog entry in PROJECT_STATE.md; copy cannot fix a positioning gap.
4. **Localization subagent returns over-budget or claim-inflated text.** Reject and rerun with
   the specific violations quoted. Never trim a returned translation yourself
   mid-word/mid-grammar in a language you are not validating against the locale keyword map —
   re-delegate with the constraint made explicit.
5. **Handover fails `07-checklists/handover-validation.md`.** Fix the handover, not the
   checklist. The usual causes: missing asset paths for Blocked items (the schema wants them
   listed as blocked, not absent), unrecorded policy attestations, variant copy smuggled into
   the final-metadata section, or an empty Baseline Metrics Snapshot. Re-validate and record the
   second run.
6. **Screenshots, feature graphic, or icon missing because Prompt 11/10 was skipped.** Ship the
   copy package with the asset rows Blocked and owners named, keep the handover in Draft until
   the assets exist (or Deliver it with the Blocked rows explicit if the operator accepts the
   risk — never invent a status outside the canonical lifecycle), and add a loop-back note per
   `01-core/PROJECT_LIFECYCLE.md` — Prompt 15 will refuse to pass verification until the assets
   exist, which is the correct enforcement point.
7. **The audit's review themes table is empty (new app, no reviews).** Review-Response Readiness
   still gets populated: write templates for the category's predictable complaint classes per
   `04-aso/workflows/ratings-reviews.md` (crashes, pricing, missing feature), note "no live
   reviews at baseline", and set the SLA/owner anyway — the section exists to be ready before
   the first review lands, not after.
