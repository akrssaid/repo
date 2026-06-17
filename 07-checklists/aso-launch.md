# ASO Launch Checklist

Gate for **P7 (ASO & Store Assets) exit**, and re-run **before any store listing publish or major
listing update** in any later phase. This checklist verifies that store metadata, visual assets,
localization, and policy posture are complete, accurate, and within each store's hard limits
(Google Play, Huawei AppGallery, RuStore — limits per `04-aso/stores/`), and that the ASO handover
is ready for the release agent. A listing that fails store validation or triggers a policy review
costs days of launch delay; this gate exists to make that impossible.

> **Run when:** End of P7 after `01-core/prompts/14-store-assets.md`; re-run before every listing
> publish.
> **Run by:** The ASO agent (self-gate), then the release agent before consuming
> `docs/factory/handovers/ASO_HANDOVER_v<N>.md`.
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification
> (e.g. "Huawei items N/A — Google Play only, per PROJECT_STATE Decisions").
> Record the completed copy as `docs/factory/reports/aso-launch-YYYY-MM-DD.md`.

"Listing source" = the instantiated `09-templates/store-listing.md` in the ASO handover. Apply
store groups per the distribution decision in `docs/factory/PROJECT_STATE.md`.

## Metadata

- [ ] Every text field is within its store's character limit, with the actual count recorded next
      to each field in the listing source. Verify per `04-aso/stores/google-play.md` (title 30,
      short description 80, full description 4000), `04-aso/stores/huawei-appgallery.md`, and
      `04-aso/stores/rustore.md`; count with
      PowerShell `("<text>").Length` per field and record: field → count/limit.
- [ ] Primary keyword (from `docs/factory/reports/KEYWORD_MAP.md`, per
      `09-templates/keyword-map.md`) appears in the app title. Verify: string match against the
      keyword map's #1 ranked keyword.
- [ ] No competitor names or third-party trademarks in title, descriptions, or keywords. Verify:
      grep the listing source for each competitor name listed in the
      `04-aso/workflows/competitor-analysis.md` output; zero hits.
- [ ] Every claim in the descriptions is true of the **current release build** (features, "no
      ads"/pricing claims, performance claims). Verify: map each claim sentence to a feature in
      `docs/factory/PROJECT_STATE.md` feature inventory; no claim references deferred work.
- [ ] No prohibited metadata patterns: emoji/decorative unicode in title (Play), "best/#1" style
      unsubstantiated superlatives, incentivized-review language. Verify: manual scan against
      `08-knowledge/stores/policy-landmines.md` metadata section.
- [ ] What's-new / release-notes text present for each store in scope and within its limit (Play:
      ≤ 500 chars per locale). Verify: the listing source has a what's-new entry per store × locale;
      count each with PowerShell `("<text>").Length` and record field → count/limit.

## Assets

- [ ] App icon 512×512 px PNG present in `docs/factory/assets/store/<store>/` (per the canonical
      store-asset tree). Verify:
      `Test-Path` plus dimensions via
      PowerShell `Add-Type -AssemblyName System.Drawing; [System.Drawing.Image]::FromFile("<path>").Size`.
- [ ] Icon passes the 48px legibility test per `04-aso/workflows/icon-optimization.md`: downscale
      to 48×48 and confirm the silhouette/glyph is still recognizable; archive the downscaled
      sample beside the source. Verify: file exists and reviewer verdict recorded.
- [ ] Feature graphic 1024×500 px present (Google Play). Verify: `Test-Path` + dimension check as
      above. N/A for stores that do not use it (justify per store list).
- [ ] Screenshot sets complete per store per claimed locale: minimum counts and exact dimensions
      verified (Play: 2–8 phone screenshots, each side 320–3840 px, 16:9 or 9:16 recommended;
      AppGallery and RuStore per their `04-aso/stores/` docs). Verify: enumerate
      `docs/factory/assets/store/<store>/<locale>/` and dimension-check each file;
      record store → locale → count.
- [ ] Screenshot captions/overlay copy match the per-screenshot assignments in
      `docs/factory/reports/KEYWORD_MAP.md` from `04-aso/workflows/screenshot-optimization.md`.
      Verify: compare each caption against the keyword map row for that screenshot slot.
- [ ] Screenshots show the **redesigned** UI (post-P6), both as rendered — no mockups of
      unimplemented features. Verify: spot-compare against
      `docs/factory/assets/screenshots/redesigned/`.

## Localization

- [ ] Every locale claimed in the listing is 100% complete: all metadata fields + full screenshot
      set + store-required graphics exist for that locale. Verify: per-locale completeness table
      built from the `04-aso/workflows/localization.md` locale list; no partial rows.
- [ ] Native-quality check done for Tier-1 locales (as designated in the localization workflow
      output): metadata reviewed for fluency, character-limit compliance re-verified post-review
      (translations expand), and culturally wrong idioms removed. Verify: review note per Tier-1
      locale recorded in the ASO handover.
- [ ] In-app language support matches claimed locales, or the listing does not imply in-app
      localization it lacks. Verify: compare `app/src/main/res/values-*` locales against the
      listing locale list; discrepancies justified.

## Policy

- [ ] Policy landmine sweep clean per `08-knowledge/stores/policy-landmines.md`: every landmine
      category in that doc checked against this app and listing, result recorded per category.
      Verify: the sweep table is attached to this report with zero unresolved hits.
- [ ] Play Data safety form content matches the actual SDK inventory from
      `02-engineering/04-sdk-inventory.md` output (every SDK's data collection/sharing declared;
      nothing declared that is not collected), mapped per `04-aso/workflows/data-safety-listing.md`.
      Verify: row-by-row diff of the Data safety draft vs the SDK inventory table, using the
      data-type → declaration mapping in `04-aso/workflows/data-safety-listing.md`; every collected
      type is declared and nothing extra is.
- [ ] Privacy policy URL is live (HTTP 200) and its content accurately covers current data
      practices including all third-party SDKs. Verify:
      PowerShell `(Invoke-WebRequest "<url>").StatusCode` = 200; content review note recorded.
- [ ] Ads / in-app purchase declarations match the build (contains-ads flag, IAP flag, families
      policy applicability). Verify: cross-check against monetization SDKs in the SDK inventory.

## Store Configuration

- [ ] Category and tags recorded with rationale per store. Verify: the ASO handover's Category &
      tags rationale block names the chosen primary (and secondary, where the store allows it)
      category and tag set per store, each with a one-line rationale tied to the keyword map.
- [ ] Content-rating questionnaire completed per store (Play IARC, AppGallery, RuStore). Verify:
      each store's questionnaire answers are recorded/attached and consistent with the app's actual
      content; the resulting rating is noted per store, or N/A with justification for stores in
      scope that do not gate on it.
- [ ] Store contact email and privacy-policy URL present and correct per store. Verify: the
      contact email is set in each console listing and the privacy-policy URL field is populated
      (URL liveness checked in the Policy group above).
- [ ] In-app review API integrated per the factory doctrine
      (`04-aso/workflows/ratings-reviews.md`), or N/A with justification. Verify:
      `rg "ReviewManager|requestReviewFlow|launchReviewFlow" --type kotlin --type java` shows the
      Play In-App Review (or store-equivalent) integration, or record a one-line N/A justification.

## Launch & Experiments

- [ ] Pre-publish baseline metrics captured in the ASO handover's Baseline Metrics Snapshot table
      (impressions, CVR by channel, installs, rating avg/count, head-term ranks est.). Verify: the
      table exists with concrete numbers and a capture date — not blanks — so post-release deltas
      are measurable.
- [ ] Staged-rollout plan stated (the listing/launch-timing side): the rollout percentages, dwell,
      and listing-publish timing are recorded in the handover's launch-timing block, consistent with
      `10-releases/release-workflow.md` stage 4. Verify: the launch-timing block exists with explicit
      numbers and references the canonical rollout gates.
- [ ] First listing experiment queued with an `EXP-NNN` id in the ASO handover's EXP-queue rows.
      Verify: at least one EXP row exists (variant description, hypothesis, metric, id) ready to be
      consumed by `docs/factory/reports/EXPERIMENT_LOG.md`, or N/A with justification if
      experiments are out of scope for this store.

## Handover

- [ ] `docs/factory/handovers/ASO_HANDOVER_v<N>.md` validates against
      `05-handover/ASO_HANDOVER_SCHEMA.md` and has passed
      `07-checklists/handover-validation.md` (report exists in `docs/factory/reports/`).
- [ ] Publish runbook present in the ASO handover: ordered console steps per store, rollback note
      for listing changes, and owner of the publish action. Verify: section exists and steps are
      executable without questions.

## Documentation

- [ ] Every finding from `docs/factory/audits/ASO_AUDIT.md` is Closed or Deferred with reason —
      none left open. Verify: scan the audit's findings table.
- [ ] `docs/factory/CHANGELOG.md` updated with P7 asset/metadata production entries and this gate
      run. Verify: read the changelog.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / FAIL |
| Stores in scope | |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |

On PASS: advance the phase tracker in `docs/factory/PROJECT_STATE.md` to P8 and log this report in
`docs/factory/CHANGELOG.md`. Re-run this checklist in full before the actual publish action.
