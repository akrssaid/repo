# Google Play — Store Operating Guide

This guide is the factory's authoritative reference for operating a Google Play listing: metadata
fields and limits, what the search index actually reads, graphic asset specifications, ranking
signals, Play-only features (experiments, custom store listings, LiveOps), the review process, and
the maintenance rules (Data safety, target API) that keep an app visible and publishable. Used by
prompts `01-core/prompts/12-aso-audit.md`, `13-keyword-research.md`, and `14-store-assets.md`.

> **Standing instruction:** store specs change. Values below are complete and current as of
> 2026-06-10, but always **verify limits in the store console (Play Console) before submission**.

---

## 1. Metadata fields & limits

| Field | Limit | Indexed for search | Notes |
|---|---|---|---|
| App title | 30 chars | **Yes — strongest signal** | Lead with the highest-value relevant keyword; brand + category descriptor pattern (e.g. "PDF Reader — Solar Docs"). No emoji, no ALL CAPS words, no rank/price claims ("#1", "best", "free") — metadata policy violation. |
| Short description | 80 chars | **Yes — strong** | First thing read on the listing; must contain top 1–2 keywords *and* a conversion hook. Rewrite as a benefit statement, not a feature list. |
| Full description | 4000 chars | **Yes — moderate** | See §1.1 keyword density guidance. First ~250 chars show before "Read more" — front-load value proposition and primary keywords. |
| Developer name | 50 chars | Yes — weak | Keep stable; changing it can disrupt brand searches. |
| What's new (release notes) | 500 chars | Weakly / inconsistently | Do not keyword-stuff; use it to drive update conversion — see §7. |
| App category | 1 primary | Indirect (browse/charts) | Choose where competition is winnable, not just where it "fits". |
| Tags | up to 5 | Indirect | Set in Play Console; influences peer grouping for charts and similar-apps. |
| Contact email / website / privacy policy URL | required | No | Privacy policy URL is mandatory for all apps. |

### 1.1 Full description keyword usage

- Keyword density follows the **canonical rule in `08-knowledge/aso/ranking-factors.md`** (2–4
  natural uses of a primary keyword per 4000-char description, hard cap 5). Do not substitute a
  different number here — that doc is the single source.
- Use semantic variants and plurals; Play's index handles stemming, but exact-match occurrences
  of the title/short-description keywords reinforce relevance.
- **Never** paste keyword lists, comma-separated keyword blocks, or competitor app names. Play
  flags keyword stuffing as a metadata policy violation and it can trigger listing rejection.
- Structure: hook paragraph (≤250 chars) → 3–5 benefit blocks with ✓/– bullets → feature detail →
  social-proof/permissions transparency → closing CTA. Bullets render as plain text but aid scan.

## 2. Graphic asset specs

> **This section is the AUTHORITATIVE source for Play screenshot specifications.** Prompt 11
> (`01-core/prompts/11-screenshot-generation.md`) cites these numbers; it must not restate a
> different set. Key facts: **min side 320 px, max side 3840 px**. The **1080 px+** figure is a
> **featuring-eligibility** threshold, **not a hard floor** — screenshots between 320 px and
> 1080 px upload and publish fine, they are just not eligible for certain featuring slots.

| Asset | Spec | Required | Notes |
|---|---|---|---|
| App icon | 512×512 PNG (32-bit), ≤ 1 MB | Yes | No transparency in the visible artwork edge; Play applies its own corner mask. Must match the launcher icon — mismatch is a rejection cause. Pair with `04-aso/workflows/icon-optimization.md`. |
| Feature graphic | 1024×500 JPG/PNG, ≤ 15 MB | Yes (required if promo video set; effectively always provide) | Shown in search/featuring placements and behind the video play button. No fine print — it renders small. |
| Phone screenshots | 2–8 images; aspect between 16:9 and 9:16; **min side 320 px, max side 3840 px**; JPG/PNG ≤ 8 MB each | Yes (min 2) | **1080 px+ unlocks featuring eligibility — it is not a publish floor.** First 2–3 carry almost all conversion weight — order per `04-aso/workflows/screenshot-optimization.md`. |
| 7-inch tablet screenshots | up to 8, same constraints | For tablet featuring | Provide real tablet layouts, not stretched phone shots. |
| 10-inch tablet screenshots | up to 8, same constraints | For tablet featuring | Required (1920 px+ recommended) for "Designed for tablets" placements. |
| Promo video | YouTube URL (public or unlisted, monetization ads off, not age-restricted) | Optional | Autoplays muted in many placements; first 5 seconds must work without sound. |

## 3. Ranking signals (what the algorithm weighs)

In approximate order of leverage for a small publisher — full treatment in
`08-knowledge/aso/ranking-factors.md`:

1. **Keyword relevance** — title > short description > full description; plus user search-query →
   install behavior ("did people searching X install this app and keep it?").
2. **Install velocity** — recent install trend matters more than lifetime total; a growing app
   outranks a larger stagnant one for contested terms.
3. **Retention & engagement** — D1/D7/D30 retention from Play's own telemetry feed ranking and
   featuring eligibility. This is why doctrine says relevance before volume.
4. **Ratings — quality and recency** — recent rating average is weighted above lifetime; respond
   to reviews (responses measurably lift re-ratings) and use the in-app review API at positive
   moments, never gated or incentivized.
5. **Technical vitals — visibility thresholds.** Play's bad-behavior thresholds: user-perceived
   **ANR rate ≥ 0.47%** or user-perceived **crash rate ≥ 1.09%** (overall, or 8%/8% on a single
   device model) reduce discoverability and can trigger a warning shown on the listing. Check
   Android vitals before every ASO push; fixing vitals is an ASO action.
6. **Conversion rate** of the listing itself — higher impression→install CVR reinforces rank.

## 4. Play-specific features to exploit

| Feature | What it does | How the factory uses it |
|---|---|---|
| Store listing experiments (A/B) | Test icon, feature graphic, screenshots, descriptions; **up to 3 variants tested against the current listing** (the current listing is the control, not one of the 3) | One variable at a time. Run ≥ 7 days **and** until Play reports a confident result — as a floor, do not conclude under ~1,000 installs per arm for asset tests; small effects need more. Design and queue these via `04-aso/workflows/experiment-program.md`; log every experiment in `docs/factory/reports/EXPERIMENT_LOG.md` and `docs/factory/CHANGELOG.md`. |
| Custom store listings | Different listing per country, region, or pre-registration/inactive-user audience (up to 50) | Use for top locales from `04-aso/workflows/localization.md` and for markets where a different hero use-case converts (e.g. offline mode). |
| LiveOps / promotional content | Time-boxed event/offer cards surfaced on the listing and across Play | Use at major releases and seasonal moments; needs strong creative, drives both new installs and re-engagement. |
| Pre-registration | Listing live before the app | Relevant only for new launches; banks day-1 install velocity. |
| In-app review API | Rating prompt without leaving the app | Trigger after a success moment; never after a crash or on first launch. |

## 5. Review process & common rejection causes

- Typical review time: a few hours to ~3 days for updates; **up to 7+ days** for new apps, new
  developer accounts, or apps with sensitive permissions. Plan release dates accordingly
  (`10-releases/release-workflow.md`).
- Use staged rollout (e.g. 10% → 50% → 100%) so a policy or vitals problem doesn't hit everyone.

| Rejection cause | Prevention |
|---|---|
| Metadata policy: keyword stuffing, "best/#1/free" claims, emoji in title | Follow §1; final copy review against policy before upload. |
| Data safety form doesn't match observed data collection | §6 procedure; re-run after any SDK change. |
| Sensitive permission without approved declaration (SMS, call log, all-files access, accessibility, background location) | `02-engineering/06-manifest-audit.md` must justify or remove each; file the declaration form with an in-app demo video where required. |
| Screenshots/description misrepresent functionality | Assets must show real app UI; no simulated features. |
| Target API level below policy floor | §6 target-API rule. |
| Broken functionality / crash on review device | `01-core/prompts/15-final-verification.md` gate before submission. |
| Privacy policy URL missing, dead, or not mentioning the app | Verify URL resolves and names the app and developer. |
| Families/ads policy mismatch (if children in target audience) | Declare audience honestly; ads SDKs must be self-certified for families if applicable. |

## 6. Maintenance rules (standing)

1. **Data safety form must match the SDK inventory.** After any dependency change, re-run
   `02-engineering/04-sdk-inventory.md`, map each SDK's data collection/sharing against the
   current Data safety declaration, and resubmit the form if anything changed. A mismatch is a
   removable offense and a common silent rejection cause. Record the reconciliation date in
   `docs/factory/PROJECT_STATE.md`. **The full Data-safety form procedure (how to fill each
   section from the SDK inventory) lives in `04-aso/workflows/data-safety-listing.md`** — follow
   that workflow rather than ad-hoc filling.
2. **Target API policy.** Play requires new apps and updates to target an API level within one
   year of the latest major Android release; the deadline cycles **every August**. State both
   horizons explicitly:
   - **Enforced floor NOW (June 2026): `targetSdk` 35.** Apps targeting below API 35 cannot ship
     updates and become invisible to new users on newer devices today.
   - **Floor at the next deadline (August 2026): `targetSdk` 36.** API 36 becomes the enforced
     floor at the August 2026 deadline — plan the bump before then, do not wait for enforcement.

   Track the migration in `02-engineering/migrations/android-sdk-migration.md`.
3. **Account hygiene.** Inactive developer accounts and stale apps are subject to closure/removal
   policies — every published app should get at least one update per year.

## 7. What's new field (500 chars)

- Indexed only weakly — do not waste it on keywords. Its job is **update conversion**: the text
  shows to existing users deciding whether to update and to prospects checking app momentum.
- Lead with the single most user-visible improvement in plain language; 2–4 short lines; mention
  fixes generically ("stability improvements") unless a fix is itself a selling point.
- Never ship an empty or boilerplate-only ("bug fixes") what's-new on a release that contains
  visible features — it discards a free conversion touchpoint. Template lives in
  `09-templates/store-listing.md`.

## 8. Pre-submission checklist hook

Before any Play submission, run `07-checklists/aso-launch.md` and confirm: all §1 limits
re-verified in Play Console, §2 assets uploaded at exact specs, Data safety reconciled (§6.1),
vitals below §3.5 thresholds, and the listing copy logged in `docs/factory/CHANGELOG.md`.
