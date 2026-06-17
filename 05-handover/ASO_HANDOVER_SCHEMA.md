# ASO Handover Schema

This schema defines the contract for the ASO→operator/release handover: the document produced by
`01-core/prompts/14-store-assets.md` (lifecycle phase P7) and consumed by the human operator who
publishes store listings, plus the release prompts (`01-core/prompts/15-final-verification.md`,
`01-core/prompts/16-release-preparation.md`) that read it as input. The consumer pastes copy into
store consoles and uploads assets verbatim; nothing may require judgment, rewriting, or trimming
at publish time. Character counts, dimensions, paths, and policy attestations must therefore be
final and verified in the document itself.

## File Naming & Versioning

- Instance path: `docs/factory/handovers/ASO_HANDOVER_v<N>.md` in the TARGET app repo.
- `<N>` starts at 1; every revision bumps N. Never edit a handover after its status reaches
  Accepted — issue v(N+1) instead. After acceptance only `## Rejection Log` and
  `## Validation Appendix` may be appended (per `01-core/CONVENTIONS.md`).
- Status lifecycle (recorded in the Handover Header): Draft → Delivered → Accepted | Rejected
  — there is **no "Partial"** state (canonical scale in `01-core/CONVENTIONS.md`).
- Must pass `07-checklists/handover-validation.md` before any copy is entered into a store console.

## Required Sections

The document must contain exactly these 12 H2 sections, in this order. A missing or empty section
is an automatic validation failure. Two further sections are **whitelisted but not required** and
may be appended after delivery without breaking immutability: `## Rejection Log` (consumer, on
reject) and `## Validation Appendix` (gate re-run evidence). No other section may be added.

### 1. Handover Header

A key/value table with all of: App name, Package ID, App version this listing targets
(versionName/versionCode), Handover version (vN), Producer (`01-core/prompts/14-store-assets.md`),
Date produced (absolute), Factory version, Status (Draft / Delivered / Accepted / Rejected),
Supersedes (vN-1 or "—"), Stores covered (subset of Google Play / Huawei AppGallery / RuStore).

### 2. Strategy Summary

Prose (10–25 lines) covering: the positioning thesis (the one sentence this listing must make
true in the user's mind); primary keyword clusters (3–6 clusters with head terms, sourced from
the `13-keyword-research.md` output); competitor context (2–4 named competitors and the
differentiation angle this listing exploits, per `04-aso/workflows/competitor-analysis.md`).
This section explains *why* the copy reads the way it does, so a future cycle can revise
deliberately instead of guessing.

### 3. Per-Store Listings

One H3 subsection per store in Stores covered. Each subsection lists **every** metadata field that
store accepts, with final copy and the character count shown against the store's limit in the
heading or label — format `Field (used/limit)`, e.g. `Title (28/30)`. Field sets and limits per
store are defined in `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`, and
`04-aso/stores/rustore.md`. At minimum:

- **Google Play**: Title (30), Short description (80), Full description (4000).
- **Huawei AppGallery**: App name (64), Brief introduction (80), Description (8000).
- **RuStore**: App name (50), Short description (80), Description (4000).

Beyond the headline text fields, each store subsection must also state the non-copy listing
settings the operator will be asked for: category, tags/keywords field where present in console
(AppGallery and RuStore expose a keyword/tags input where present in console; Google Play has no
keyword field), content rating, contact email, and privacy policy URL. Fields without character
limits are labeled `(n/a)` rather than omitted, so the operator never meets a console field the
document is silent about.

Copy is **final**: exactly one value per field — no alternatives, no bracketed options, no
"choose one". This handover carries no copy variants. Variants are owned elsewhere: candidate copy
lives in the `reports/STORE_LISTING_<store>_<locale>.md` files, and any planned A/B test of a
variant is queued as an `EXP-NNN` row in the Experiment Queue (§11), consumed by
`09-templates/experiment-log.md`. The console always receives the single final string from this
section.

### 4. Category & Tags Rationale

The deliberate reasoning behind the store taxonomy choices the operator will set — so a future
cycle can revisit them on evidence, not guess. One block per store:

| Field | Choice | Rationale (why this beats the alternatives) |
|---|---|---|
| Primary category | … | competitor density, browse-traffic estimate |
| Secondary category (Play) / app type | … | … |
| Tags / keyword field (where present in console) | … | which clusters they reinforce |

State the runner-up category considered and why it was rejected. This section is rationale only;
the exact values the operator types still live verbatim in the §3 Per-Store Listings blocks.

### 5. Keyword Map Reference

A link to the full keyword research report (`docs/factory/reports/`, built from
`09-templates/keyword-map.md`) plus an inline slot-mapping table showing where each priority
keyword landed:

| Keyword | Cluster | Store | Placement (Title / Short desc / Full desc ×N / not placed) |
|---|---|---|---|

Every keyword designated Priority in the keyword report must appear in this table, including those
deliberately not placed (with reason).

### 6. Asset Package

Per store, a manifest table of every visual asset to upload:

| Field | Content requirement |
|---|---|
| Asset | icon / feature graphic / screenshot N / promo video |
| Path | Repo-relative path under `docs/factory/assets/store/<store>/` |
| Dimensions | Exact pixels, e.g. 1024×500 — must match the store spec in `04-aso/stores/<store>.md` |
| Locale variants | Locales with a localized variant, with paths, or "all locales share" |
| Notes | Caption text shown on the screenshot, device frame used |

Screenshot sets must reflect the `04-aso/workflows/screenshot-optimization.md` workflow output and
the app version stated in the header (no screenshots of older UI). Dimensions defer to the store
spec in `04-aso/stores/<store>.md` (Play min side 320px / max side 3840px; never restate numbers).

### 7. Localization Package

A table of every locale the listing claims:

| Locale | Stores | Listing fields status (Complete / Partial: missing X) | Screenshot set status | Translation source (native / machine+review / machine) |
|---|---|---|---|---|

Only locales marked Complete in all claimed fields may be published. Per
`04-aso/workflows/localization.md`, partial locales stay in this table as roadmap, flagged
explicitly in this column — never silently half-published. (A per-locale "Partial" status here is a
field-completeness note only; it is **not** a handover Status — the handover itself is only ever
Draft / Delivered / Accepted / Rejected.)

### 8. Policy Compliance Statement

An explicit attestation block: copy and assets checked against
`08-knowledge/stores/policy-landmines.md` (state the date checked and which sections apply); a
claims table listing every factual or comparative claim made in any listing field ("works
offline", "no ads", "#1 …") with its verification ("verified true: feature exists since v3.0" /
"removed"); confirmation that no prohibited content patterns (misleading screenshots, incentivized
review prompts, undisclosed data collection claims) are present. Unverifiable claims must be
removed from copy before delivery, not annotated.

### 9. Baseline Metrics Snapshot

A **mandatory** table capturing the store's pre-listing-change state, so post-publish lift is
measurable against a dated baseline rather than memory. If this is a brand-new listing, state
"first listing — no baseline" in each cell, but the table itself is still required.

| Metric | Value | As-of date | Source console |
|---|---|---|---|
| Store-listing impressions (per week) | … | … | Play Console / AppGallery Connect / RuStore |
| Conversion rate by channel (store search / explore / third-party referrer) | … per channel | … | … |
| Installs (per week) | … | … | … |
| Rating — average | … | … | … |
| Rating — count | … | … | … |
| Head-term ranks (estimate, per cluster head, dated) | e.g. "offline notes ≈ #14 (2026-06-17)" | … | rank-tracker / console search |

Head-term ranks are explicitly **estimates** and must carry the date they were observed. Conversion
rate is broken out **by channel** (search vs explore vs referrer), not a single blended number.

### 10. Review-Response Readiness

The pre-written response kit the operator uses to reply to reviews during launch, per
`04-aso/workflows/ratings-reviews.md`. One reply template per top complaint theme (mined in the ASO
audit and the DESIGN handover's User Insight section), plus the owner and SLA for responding.

| Complaint theme | Reply template (paste-ready, ≤ store reply limit) | Owner | Response SLA |
|---|---|---|---|

State the single SLA owner (a named person/role) accountable for review responses in the launch
window, and the cadence (e.g. "1–2★ within 24 h, 3★ within 72 h, first 14 days").

### 11. Experiment Queue

The ordered backlog of post-publish experiments, each with a stable `EXP-NNN` ID (per the
`01-core/CONVENTIONS.md` registry) **consumed by `09-templates/experiment-log.md`**. Play store
listing experiments allow up to 3 variants vs the current listing (`04-aso/stores/google-play.md`
authoritative — cite, do not restate caps here for other stores).

| EXP-ID | Hypothesis | Variable changed (variant lives in STORE_LISTING file) | Success metric + target | Store | Priority |
|---|---|---|---|---|---|

Every variant referenced here is a STORE_LISTING candidate, never an inline copy alternative in §3.
The IDs assigned here are the same IDs the experiment log later reports outcomes against.

### 12. Launch Timing & Rollout

The store-side timing plan the operator executes (distinct from the app artifact's staged rollout,
which the RELEASE handover owns). Cover:

- **Pre-registration**: yes/no per store; if yes, target go-live date and reward (if any).
- **Staged listing rollout**: whether the listing/update is published to all users at once or
  staged, and the schedule.
- **Per-store review-time budget**: the review/approval window to budget per store before assets
  go live (defer to the store guide in `04-aso/stores/<store>.md` for current durations — cite,
  do not invent), so the operator schedules submission early enough.

| Store | Pre-registration | Rollout schedule | Review-time budget (per store guide) |
|---|---|---|---|

### 13. Publish Runbook

One H3 per store: exact, numbered console steps the human operator performs — where to navigate,
which field receives which section of this document, upload order for assets, locale switching
steps, and what to save/submit at the end. Mark every step that only a human can perform
(account login, payment profile, final submit button). The runbook must be executable by an
operator who has console access but did not read the rest of the document.

## Required Deliverables

1. The handover document at `docs/factory/handovers/ASO_HANDOVER_v<N>.md`.
2. Every asset listed in Asset Package, present at its stated path with its stated dimensions.
3. The keyword report referenced in Keyword Map Reference, present under `docs/factory/reports/`.

## Validation Rules (mechanically checkable)

| # | Rule |
|---|---|
| V1 | All 13 required H2 sections present, in order, non-empty (only `## Rejection Log` and `## Validation Appendix` may appear additionally) |
| V2 | Handover Header contains all ten fields; Stores covered non-empty; Status ∈ {Draft, Delivered, Accepted, Rejected} (no "Partial") |
| V3 | Every listing field for every store and every locale shows a character count in `(used/limit)` form, and used ≤ limit for all — counts recomputed mechanically, zero violations |
| V4 | Every store in Stores covered has a Per-Store Listings subsection, a Category & Tags Rationale block, an Asset Package table, a Launch Timing row, and a Publish Runbook subsection |
| V5 | Every asset path in Asset Package exists on disk, and stated dimensions match the store spec in `04-aso/stores/<store>.md` |
| V6 | Every locale claimed in Localization Package as Complete has the full field set present in Per-Store Listings (or locale-variant blocks) and a complete screenshot set in Asset Package |
| V7 | Every claim in listing copy appears in the Policy Compliance claims table with verification "verified true" (no unverifiable claims remain in copy) |
| V8 | Every Priority keyword from the referenced keyword report appears in the slot-mapping table; the report file exists |
| V9 | Baseline Metrics Snapshot is present with every row populated (a value, or "first listing — no baseline"); head-term rank rows carry an as-of date |
| V10 | Review-Response Readiness has ≥1 reply template, a named SLA owner, and a response cadence |
| V11 | Every Experiment Queue row has a unique `EXP-NNN` ID and references a STORE_LISTING-owned variant (no inline copy variant anywhere in §3) |
| V12 | No unresolved references: every linked report/asset path exists; no placeholder tokens (`{{...}}`), no "TBD"/"TODO" anywhere |

## Acceptance Criteria (consumer's bar)

The operator (or release agent) accepts only if all hold; otherwise rejects:

1. **The paste test**: every field can be copied into the console verbatim with zero edits,
   trims, or decisions.
2. The runbook can be executed start-to-finish by someone who reads only the runbook, with each
   step pointing at the exact copy block or asset to use.
3. Counts and dimensions are trustworthy on spot-check: re-counting any 3 fields and re-measuring
   any 2 assets matches the stated values.
4. The compliance statement leaves no claim unaccounted for — nothing in the copy makes the
   operator policy-nervous without a verification row.
5. The Baseline Metrics Snapshot gives a dated starting point, and the Review-Response kit + SLA
   owner mean the operator can field launch-window reviews without improvising.
6. All Validation Rules V1–V12 pass on the consumer's re-run of
   `07-checklists/handover-validation.md`.

## Rejection Protocol

1. The consumer appends a `## Rejection Log` section to the handover file (one of only two
   permitted post-delivery appends, the other being `## Validation Appendix`), one row per reason:

   | Date | Rejected by | Rule / criterion violated | Detail |
   |---|---|---|---|

2. The consumer sets Status to Rejected; no copy from a rejected handover may be entered into any
   store console.
3. The producer issues `ASO_HANDOVER_v<N+1>.md` addressing every logged reason, with Supersedes
   set to vN. The rejected file is never deleted or edited further.
4. Both events are recorded in `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md`.

## Producer & Consumer Workflow

1. **Producer** (prompt 14) drafts with Status: Draft, assembling inputs from the ASO audit
   (`docs/factory/audits/ASO_AUDIT.md`), the keyword research report (prompt 13), and the asset
   workflows in `04-aso/workflows/`. Character counts are computed mechanically (count the
   string, don't estimate) for every field in every locale.
2. Producer self-runs `07-checklists/handover-validation.md` (rules V1–V12), including a disk
   check of every asset path and dimension, and a mechanical recount of every character count.
3. Producer sets Status: Delivered, records delivery in `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md`.
4. **Consumer** (operator, or prompts 15/16 reading it as input) re-runs the gate and performs
   the spot-checks in Acceptance Criterion 3 before touching a console.
5. On accept: consumer sets Status: Accepted and executes the Publish Runbook store by store,
   recording publish dates in PROJECT_STATE. On reject: follow the Rejection Protocol below.
6. Post-publish copy changes (an A/B winner, a policy takedown fix) are a new cycle: producer
   issues v(N+1); consoles are never edited ahead of the document of record.

## Common Failure Modes

| Failure | Symptom downstream | Prevention |
|---|---|---|
| Estimated character counts | Console rejects the paste at limit; operator trims ad hoc, destroying keyword placement | V3: counts shown per field, mechanically verified |
| Screenshots of pre-redesign UI | Listing contradicts the app; conversion drops, policy risk | Asset Package must match the app version in the header; spot-check at acceptance |
| Unverifiable superlatives ("#1 notes app") | Store rejection or policy strike | V7: every claim verified true or removed before delivery |
| Locale claimed Complete with machine-only long description | Bad-quality listing live in a market nobody re-reads | V6 + Translation source column makes quality level explicit |
| Runbook says "fill in the listing" | Operator improvises field mapping; copy lands in the wrong field | Runbook steps must name the exact §3 block per console field |

## Example Skeleton

```markdown
# ASO Handover — NotesPro v3.2.0

## 1. Handover Header
| Field | Value |
|---|---|
| App | NotesPro (com.example.notespro) |
| App version | 3.2.0 (320) |
| Handover version | v1 |
| Producer | 01-core/prompts/14-store-assets.md |
| Date produced | 2026-06-18 |
| Factory version | 1.1.0 |
| Status | Delivered |
| Supersedes | — |
| Stores covered | Google Play, RuStore |

## 2. Strategy Summary
Positioning thesis: the fastest offline note app for field work. Clusters: offline notes, ...

## 3. Per-Store Listings
### Google Play (en-US)
Title (29/30): `NotesPro: Offline Field Notes`
Short description (70/80): `Capture notes in seconds, fully offline. Sync when you're back online.`
Full description (664/4000): `NotesPro is the fastest way to capture notes in the field … never lose a field note again.`
Category (n/a): Productivity · Content rating (n/a): Everyone · Contact email (n/a): support@notespro.app · Privacy policy (n/a): https://notespro.app/privacy

## 4. Category & Tags Rationale
### Google Play
| Field | Choice | Rationale |
|---|---|---|
| Primary category | Productivity | higher browse traffic than Tools for note apps; runner-up Tools rejected (commodity, lower CVR) |

## 5. Keyword Map Reference
Report: docs/factory/reports/KEYWORD_MAP_2026-06-17.md
| Keyword | Cluster | Store | Placement |
|---|---|---|---|
| offline notes | Offline | Google Play | Title, Full desc ×3 |

## 6. Asset Package
### Google Play
| Asset | Path | Dimensions | Locale variants | Notes |
|---|---|---|---|---|
| Feature graphic | docs/factory/assets/store/google-play/en-US/feature.png | 1024×500 | all locales share | tagline overlay |

## 7. Localization Package
| Locale | Stores | Fields | Screenshots | Source |
|---|---|---|---|---|
| de-DE | Google Play | Complete | Complete | machine+review |

## 8. Policy Compliance Statement
Checked against 08-knowledge/stores/policy-landmines.md on 2026-06-18 (sections: metadata, claims).
| Claim | Where | Verification |
|---|---|---|
| "fully offline" | Short desc | verified true: no network permission required since v3.0 |

## 9. Baseline Metrics Snapshot
| Metric | Value | As-of date | Source console |
|---|---|---|---|
| Store-listing impressions/wk | 12,400 | 2026-06-17 | Play Console |
| Conversion rate by channel | search 4.1% / explore 2.2% / referrer 6.0% | 2026-06-17 | Play Console |
| Installs/wk | 410 | 2026-06-17 | Play Console |
| Rating — average | 4.2 | 2026-06-17 | Play Console |
| Rating — count | 1,830 | 2026-06-17 | Play Console |
| Head-term ranks (est., dated) | "offline notes ≈ #14 (2026-06-17)" | 2026-06-17 | rank-tracker |

## 10. Review-Response Readiness
SLA owner: M. Operator. Cadence: 1–2★ within 24 h, 3★ within 72 h, first 14 days.
| Complaint theme | Reply template | Owner | SLA |
|---|---|---|---|
| "lost my notes" | "Sorry to hear that — notes save locally; please reach support@notespro.app and we'll recover them." | M. Operator | 24 h |

## 11. Experiment Queue
| EXP-ID | Hypothesis | Variable changed | Success metric + target | Store | Priority |
|---|---|---|---|---|---|
| EXP-001 | Icon variant B lifts CVR | icon (variant in STORE_LISTING_google-play_en-US.md) | store-listing CVR +0.3pp | Google Play | High |

## 12. Launch Timing & Rollout
| Store | Pre-registration | Rollout schedule | Review-time budget (per store guide) |
|---|---|---|---|
| Google Play | No | publish to 100% at submit | per 04-aso/stores/google-play.md |

## 13. Publish Runbook
### Google Play
1. [HUMAN] Sign in to Play Console → NotesPro → Grow → Store presence → Main store listing.
2. Paste Title from §3 Google Play block. ...
```
