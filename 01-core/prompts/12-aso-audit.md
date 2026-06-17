# Prompt 12 — ASO Audit

**Lifecycle phase:** P7 — ASO & Store Assets

This prompt establishes the factual baseline for all ASO work: it captures the current store
listing exactly as shipped, scores it against known ranking and conversion factors, profiles the
top keyword competitors, checks for store-policy risk, and emits a severity-ranked findings list
that Prompts 13 (keyword research) and 14 (store assets) consume. Nothing in P7 should be
changed before this audit exists — you cannot measure improvement without a baseline.

## Role

You are a **Senior ASO Specialist** with deep experience across Google Play, Huawei AppGallery,
and RuStore. You audit like an engineer: every claim in your report is backed by an observed
fact (a captured listing field, a counted character, a competitor's actual title), never by
intuition. You distinguish ranking problems (the app is not found) from conversion problems (the
app is found but not installed) and from policy problems (the app may be removed), because each
has a different fix and a different owner.

## Objective

Produce `docs/factory/audits/ASO_AUDIT.md` — instantiated from
`09-templates/aso-audit-report.md` — containing: a verbatim baseline snapshot of every targeted
store listing including a dated baseline keyword-rank capture for the head terms, a scored
assessment against `08-knowledge/aso/ranking-factors.md` across metadata keyword coverage,
conversion assets, ratings health (with a mined own-review themes table), and localization
depth, a competitor scan of the top 5 keyword competitors per store, a policy-risk check against
`08-knowledge/stores/policy-landmines.md`, a Data-safety reconciliation per
`04-aso/workflows/data-safety-listing.md`, and a numbered findings register (ASO-001…) with
severity and recommended owner-prompt. Done means Prompt 13 can start keyword research and
Prompt 14 can start asset generation — including its handover's Baseline Metrics Snapshot and
Review-Response Readiness sections — with zero additional fact-finding.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P7 entered; P6 redesign implementation complete or explicitly waived in `docs/factory/PROJECT_STATE.md` | If P6 incomplete, proceed but flag every conversion-asset finding as "re-evaluate after redesign". |
| Target store list, app IDs, listing URLs | `docs/factory/PROJECT_STATE.md` → Store Presence section | Ask the human operator for the store URLs/package name. Do not guess package names. |
| Live listing access | Public store pages (web) for each store; console access optional | Web pages suffice for the audit. If a listing is unpublished/new, record "no live listing" and audit the draft listing the operator provides; competitor scan still runs. |
| Ranking factors knowledge | `08-knowledge/aso/ranking-factors.md` | Required reading before scoring. |
| Conversion knowledge | `08-knowledge/aso/conversion-optimization.md` | Required reading before scoring conversion assets. |
| Competitor workflow | `04-aso/workflows/competitor-analysis.md` | Required reading before the competitor scan and the own-review mining (same tagging method). |
| Policy landmines | `08-knowledge/stores/policy-landmines.md` | Required reading before the policy check. |
| Data-safety workflow | `04-aso/workflows/data-safety-listing.md` | Required reading before the Data-safety reconciliation. |
| Report template | `09-templates/aso-audit-report.md` | Required; instantiate it, do not invent a structure. |
| Web access (WebFetch/WebSearch or operator-supplied captures) | Tool availability | If no web access, request the operator paste/screenshot each listing; mark provenance "operator-supplied, 2026-06-10". |

## Agent Orchestration

- **Multi-store: one subagent per store.** When more than one store is targeted, fan out exactly
  one subagent per store (Google Play, AppGallery, RuStore). Each subagent receives: the
  package/app ID, the listing URL, the scoring rubric section references, and the locale list.
  Each returns: the baseline snapshot for its store (including the keyword-rank capture), the
  own-review themes table, per-dimension scores with evidence, the top-5 competitor table for
  its store, and candidate findings (unnumbered).
- **Single store: stay single-threaded.** The audit is sequential reading and scoring;
  parallelism buys nothing.
- **The lead agent always owns:** merging store fragments, deduplicating cross-store findings,
  assigning final ASO-NNN numbers and severities (numbering must be globally sequential across
  stores, not per store), and writing the final report. Subagents never write `ASO_AUDIT.md`.
- Cap concurrent subagents at 8 per `01-core/CONVENTIONS.md`. If a subagent cannot reach its store page, it returns a partial
  with an explicit "unverified" provenance flag rather than blocking the whole audit.

## Procedure

1. **Capture the baseline snapshot per store.** For each store and each published locale, record
   verbatim into the baseline section: app title (with character count), short description /
   subtitle (with count), full description (full text), category and tags, icon (file or
   screenshot reference saved under `docs/factory/assets/aso-baseline/`), feature graphic
   (Play), screenshot set (count, order, captions, theme), promo video presence, rating average,
   rating count, recent-review sentiment (read the 10 most recent reviews; summarize themes),
   install count band as displayed, last-update date shown, and developer-reply behavior.
   Date-stamp the snapshot 2026-06-10 and record the capture method (web fetch vs
   operator-supplied). This snapshot is the before-photo — be exhaustive; it is cheap now and
   irreplaceable later.

   **Baseline keyword-rank capture (part of step 1, per store and locale).** Pick the app's 3–5
   most plausible head terms (from the listing's own title/short-description keywords and the
   category's obvious head queries). For each term, record the app's current rank — searched in
   an incognito/logged-out store session per locale (web store page in a private browser window;
   no personalized account state). Capture a table: Term | Store | Locale | Rank (est.) |
   Date | Method. Label every rank "(est.)" — unpersonalized web search approximates but does
   not equal in-app search ranking; say so in a footnote. If the app does not appear in the
   first 50 results, record ">50". This table is the before-photo for ranking work and is
   copied into Prompt 14's Baseline Metrics Snapshot — without it, no later prompt can claim a
   ranking improvement.

   Use this rubric for every 1–5 score in steps 2–5 so scores are comparable across stores and
   across audit iterations:

   | Score | Meaning |
   |---|---|
   | 5 | Best-in-category execution; no material improvement available |
   | 4 | Solid; only minor optimizations identified |
   | 3 | Functional but clearly behind the top competitors in this dimension |
   | 2 | Materially suppressing rank or conversion; needs work this cycle |
   | 1 | Absent, broken, or actively harmful (e.g., default-language stuffing, 0 screenshots) |

2. **Score metadata keyword coverage** against `08-knowledge/aso/ranking-factors.md`. For each
   locale: extract every meaningful keyword/phrase actually present in title, short description,
   and full description; assess whether the title leads with the highest-value keyword; check
   for wasted characters (filler words, repeated brand name, decorative symbols); check
   long-tail presence in the full description. Score 1–5 with a one-line justification and at
   least two cited examples from the actual text.

3. **Score conversion assets.** Against `08-knowledge/aso/conversion-optimization.md`: do the
   first three screenshots independently communicate an install reason; are captions benefit-led
   and legible at thumbnail size; does the icon survive at 48 px next to competitors; is there a
   feature graphic/video and does it help or hurt; does the listing answer the category's
   primary objection (privacy, offline, price). Score 1–5 per sub-dimension with evidence.

4. **Score ratings health — with own-app review mining.** First mine the target's own reviews
   using the same tagging method `04-aso/workflows/competitor-analysis.md` defines for
   competitor reviews: read the most recent 30–50 reviews per store (all if fewer), tag each
   with one or more theme labels (crash/bug, performance, missing feature, ads/pricing, UX
   confusion, praise themes), then aggregate into a themes table: Theme | Frequency (count of
   tagged reviews) | Representative excerpts (2–3 verbatim quotes, ≤1 line each) | Trend
   (rising/stable/fading vs older reviews). This table goes into the report's review-mining
   section and feeds Prompt 14's Review-Response Readiness section. Then score ratings health
   1–5: average and volume vs category norms; trend (are recent reviews better or worse than
   the average); whether the roadmap (P3/P4 artifacts) already addresses the top 3 complaint
   themes; developer reply rate. A 4.6 average on 40 ratings and a 4.1 on 80,000 are different
   problems — say which one you have.

5. **Score localization depth.** Which locales have full listing translations vs machine-default
   vs none; whether screenshots/captions are localized; whether the app itself is localized for
   the listing's locales (mismatch = conversion leak); compare against the locales where the
   category's demand plausibly lives. Score 1–5.

6. **Run the competitor scan** per `04-aso/workflows/competitor-analysis.md`. Identify the top 5
   competitors per store by searching the app's 3–5 most plausible head keywords and recording
   who ranks above or adjacent. For each competitor, capture into a table: app name and
   developer; exact title text and which keywords it spends its characters on; short-description
   hook; screenshot strategy (count, framing style, caption approach, what slots 1–3 sell);
   rating/volume; price/monetization model; one thing they do better than the target app; one
   exploitable weakness. End with a 5–8 line synthesis: where the listing is outgunned, where
   there is an open positioning gap.

7. **Run the policy risk check** against `08-knowledge/stores/policy-landmines.md`. Inspect the
   listing for: misleading claims or screenshots, keyword stuffing patterns that trip metadata
   policies, restricted-category trigger words, ratings/testimonial misuse,
   impersonation/brand-name risk in title or keywords, mismatch between listing claims and
   actual app behavior (cross-check the P1 discovery report `docs/factory/audits/DISCOVERY.md`).
   Each hit becomes a finding with severity Critical or High — policy findings are never Medium,
   because the downside is removal, not lost rank.

8. **Reconcile the Data-safety listing** per `04-aso/workflows/data-safety-listing.md`. Compare
   the published Data-safety / privacy declarations on each store against what the app actually
   does: the SDK inventory and permissions from `docs/factory/audits/DISCOVERY.md` and
   `audits/ENGINEERING_AUDIT.md` (ad SDKs collecting device IDs, analytics, crash reporters,
   network data flows). Record per-category: declared vs observed vs verdict
   (match / under-declared / over-declared). Under-declaration is a removal-class risk — file
   it Critical; over-declaration suppresses conversion — file it High or Medium per impact. If
   no console access exists, reconcile against the public listing's data-safety section and
   mark provenance.

9. **Compile the findings register.** Number findings ASO-001, ASO-002, … sequentially across
   all stores (ID format per `01-core/CONVENTIONS.md`). Each finding row: ID | Store(s) |
   Locale(s) | Dimension (Keywords / Conversion / Ratings / Localization / Policy / Data
   safety) | Severity (Critical / High / Medium / Low) | Evidence
   (quote or measurement) | Recommendation | Owner prompt (13, 14, or 11 for screenshot rework;
   engineering findings route to the PROJECT_STATE Backlog). Severity rubric (scale per
   `01-core/CONVENTIONS.md`): Critical =
   policy/removal risk or listing factually wrong; High = materially suppressing rank or
   conversion now; Medium = clear improvement, not urgent; Low = polish. Example row, as the
   format model:

   | ID | Store(s) | Locale(s) | Dimension | Severity | Evidence | Recommendation | Owner |
   |---|---|---|---|---|---|---|---|
   | ASO-003 | Google Play | en-US | Conversion | High | Slot-1 screenshot is the settings screen; caption "Settings" (1 word, no benefit) | Replace slot 1 with the core-payoff screen per Prompt 11 selection rules | Prompt 11 |

10. **Write the report.** Instantiate `09-templates/aso-audit-report.md` to
    `docs/factory/audits/ASO_AUDIT.md`, filling every template field: snapshot (including the
    keyword-rank capture table), scores with evidence, the own-review themes table, competitor
    tables, policy check, Data-safety reconciliation, findings register, and a 5-line executive
    summary stating the single biggest rank lever and the single biggest conversion lever. No
    empty sections — if a store wasn't audited, the section says why.

## Expected Outputs

| Artifact | Path |
|---|---|
| ASO audit report (the deliverable) | `docs/factory/audits/ASO_AUDIT.md` |
| Baseline visual captures (icon, feature graphic, current screenshots) | `docs/factory/assets/aso-baseline/<store>/<locale>/` |

## Acceptance Criteria

- [ ] Baseline snapshot is verbatim and complete for every targeted store and published locale,
      with character counts on title and short description, dated 2026-06-10 with capture
      provenance.
- [ ] Baseline keyword-rank capture exists: 3–5 head terms per store/locale, each with rank
      labeled "(est.)", date, and the incognito-search method recorded ("&gt;50" rows allowed).
- [ ] All four scoring dimensions (keywords, conversion assets, ratings health, localization
      depth) scored 1–5 per store with at least two pieces of cited evidence each.
- [ ] Own-review themes table exists per store: theme, frequency, 2–3 verbatim excerpts, and
      trend — built with the `04-aso/workflows/competitor-analysis.md` tagging method.
- [ ] Competitor table contains exactly 5 competitors per store with all columns filled, plus a
      written synthesis.
- [ ] Policy check explicitly addresses every landmine category from
      `08-knowledge/stores/policy-landmines.md` — including "checked, no issue" lines.
- [ ] Data-safety reconciliation table (declared vs observed vs verdict) is complete per
      `04-aso/workflows/data-safety-listing.md`, with under-declarations filed Critical.
- [ ] Findings numbered ASO-001… sequentially with no gaps; every finding has severity,
      evidence, recommendation, and owner prompt; every policy finding is Critical or High.
- [ ] Report conforms to `09-templates/aso-audit-report.md` structure with zero unfilled
      placeholder tokens.
- [ ] Executive summary names one biggest ranking lever and one biggest conversion lever.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: mark the P7 ASO-audit task Done in the Phase Tracker;
  add `audits/ASO_AUDIT.md` to the Artifact Index (path, version, date, status); copy all
  Critical and High findings (ID + one-line summary) into Known Risks; record the four
  dimension scores and the head-term rank baseline per store as the ASO baseline metrics so
  post-optimization re-scores have a comparison point.
- **`docs/factory/CHANGELOG.md`**: append one entry: date 2026-06-10, prompt 12, "ASO audit
  completed — N findings (C critical, H high) across S stores", path to the report. Follow
  `01-core/CHANGELOG_TEMPLATE.md` format.

## Failure Modes & Recovery

1. **Listing page unreachable or geo-blocked (common for RuStore/AppGallery from some
   regions).** Do not skip the store silently. Ask the operator for screenshots/exports of the
   listing, mark every derived fact "operator-supplied", and proceed. If nothing is obtainable,
   the store's section states exactly what is missing and the findings register gets an ASO
   finding: "baseline unobtainable — audit incomplete for <store>", severity High.
2. **No live listing exists (pre-launch app).** Pivot the audit: baseline section records
   "unpublished"; competitor scan and policy check run at full depth (they matter more
   pre-launch); scoring dimensions are scored against the draft listing if one exists, else
   marked N/A with the note that Prompt 14 creates the listing from scratch using this audit's
   competitor synthesis.
3. **Competitor identification is circular (search results dominated by irrelevant giants).**
   Re-seed with longer, more specific queries from the app's actual feature set; a top-5 of true
   keyword competitors (apps a user would install instead) beats a top-5 of category behemoths.
   Document the queries used so the scan is reproducible.
4. **Scores feel arbitrary / evidence is thin.** Stop scoring and go collect more facts —
   re-read the listing, pull 10 more reviews, check one more locale. A score without two cited
   evidences violates the acceptance criteria; never backfill justification to match a gut
   number.
5. **Subagent store fragments conflict (same finding, different severity).** The lead agent
   resolves: keep the higher severity, merge evidence from both, note both stores in the
   finding's Store column. Re-number after merging so ASO-NNN stays gap-free.
6. **Audit uncovers an app-behavior/listing mismatch that is really an engineering bug** (e.g.,
   listing promises offline mode that is broken). File it as a Critical ASO policy finding AND
   add it to the `docs/factory/PROJECT_STATE.md` Backlog routed to engineering — do not let it
   die inside the ASO report, and do not let Prompt 14 copy the claim into new listing text.
