# Store Search Ranking Factors

Evergreen reference on how app store search ranking actually works — primarily Google Play, with
Huawei AppGallery and RuStore deltas at the end. Agents consult this during P7 (ASO & Store Assets)
and whenever a metadata, rating, or vitals decision could affect organic visibility. This file is
doctrine and facts, not procedure; the step-by-step workflows live in `04-aso/workflows/`.

**Last reviewed: 2026-06-10.** Ranking systems drift continuously and Google publishes almost
nothing officially — treat every weight below as an evidence-based model, and verify
enforcement-related claims against current official policy/docs before acting on them.

---

## 1. The mental model

Play search ranking is best modeled as **relevance × quality × momentum**:

- **Relevance** — does the metadata (and review corpus) convince the index this app answers the
  query?
- **Quality** — do behavioral and technical signals (conversion, retention, vitals, ratings) say
  users who install it are satisfied?
- **Momentum** — is the app currently earning installs on this keyword at an accelerating or
  decaying rate?

No single factor wins. A perfectly keyworded listing with bad retention sinks; a beloved app with
no keyword coverage never surfaces. Optimize the whole chain.

## 2. Metadata factors (relevance)

| Field | Indexed | Relative weight | Limits (Play) | Notes |
|---|---|---|---|---|
| App title | Yes | **Highest by far** | 30 chars | Strongest single ranking lever |
| Short description | Yes | High | 80 chars | Second-strongest indexed field |
| Full description | Yes | Moderate | 4000 chars | Density and placement matter, not raw repetition |
| Developer name | Yes | Low | — | Brand queries only, effectively |
| Package name | Weak/disputed | Very low | — | Do not contort it for keywords |
| Review text | Yes (corpus-level) | Low–moderate | — | Keywords users write in reviews reinforce relevance |
| In-app events / promo content | Indirect | Low | — | Visibility, not search rank, primarily |

Rules of thumb the evidence consistently supports:

1. **Title ≫ short description > full description.** A keyword in the title outranks the same
   keyword anywhere else. Budget the 30 title characters like ad spend.
2. **Exact-match beats partial-match.** "pdf reader" in title ranks for "pdf reader" better than
   "reader of PDF files" does. Phrase order matters; adjacency matters.
3. **Position effects.** Earlier in the title > later in the title. First ~160 chars of the full
   description (above the fold, crawled reliably) > deep body text.
4. **Combinations are free.** "PDF Reader — Document Viewer" covers "pdf reader", "pdf viewer",
   "document reader", "document viewer". Engineer titles for combinatorial coverage.
5. **Full-description density (canonical rule — this file owns it).** A **primary keyword** should
   appear **2–4 times, naturally, per 4000-character description**, with a **hard cap of 5**. Beyond
   the cap, returns are zero to negative and the text starts reading as stuffing (see §6). This is
   the single definition of the keyword-density rule for the whole factory: `01-core/CONVENTIONS.md`
   records it and `04-aso/workflows/keyword-research.md`, `04-aso/workflows/localization.md`, the
   store-listing template, and prompt 13/14 all cite **this rule** rather than restate a number
   (D16). If you find a different density figure anywhere, it is stale — fix it to point here.
6. Unlike Apple's App Store, **Play has no hidden keyword field** — everything indexed is
   user-visible, so every keyword decision is also a conversion decision. See
   `08-knowledge/aso/conversion-optimization.md`.

Localization multiplies all of the above: each localized listing is indexed for that locale's
queries. Workflow: `04-aso/workflows/localization.md`.

## 3. Behavioral factors (quality + momentum)

- **Install velocity per keyword.** Installs attributed to a query are the strongest momentum
  signal for that query. Rank rises while velocity rises, decays when it stalls. This is why rank
  is a moving equilibrium, not a ladder you climb once.
- **Conversion rate per query (CTR → install).** Play measures, per search term, how often your
  result is tapped and converts versus category peers. A listing that converts above the
  keyword's expected rate gets promoted; below, demoted. **This is why CTR/conversion
  optimization IS ranking optimization** — icon, title, and rating shown in search results are
  ranking inputs, not just conversion inputs.
- **Retention as a quality signal.** D1/D7/D30 retention cohorts feed app quality. Apps that get
  installed and abandoned in 48 hours are treated as having deceived the query. Sustained D7+
  retention is the cheapest "ranking tactic" that exists: ship an app people keep.
- **Uninstall velocity.** A spike in uninstalls (especially soon after install) is a strong
  negative signal and often follows misleading metadata or aggressive ads. Fix the cause before
  touching keywords.
- **Engagement breadth.** DAU/MAU, open frequency, and Android vitals engagement proxies
  contribute; weight is lower than retention but nonzero.

## 4. Reputation factors

- **Rating magnitude**: Play displays and weights the rating users in *that country on that device
  class* gave recently — it is not a global lifetime average. Below ~4.0 both conversion and rank
  suffer materially; below 3.0 risks listing-quality interventions.
- **Recency and velocity**: a recent stream of 5★ reviews outweighs an old pile. Rating decay
  means reputation must be continuously earned — drive reviews via the in-app review API at
  success moments (doctrine in `08-knowledge/aso/conversion-optimization.md` §6).
- **Review keyword content**: review text is part of the relevance corpus. Users writing "best
  pdf scanner" in reviews measurably reinforces those terms. Never solicit specific wording —
  that crosses into incentivized-review territory (§6) — but know the effect exists.
- **Reply rate**: developer replies correlate with rating recovery (users can revise ratings
  after a reply) and signal account health.
- **Developer account standing**: policy strikes, prior removals, and associated-account flags
  suppress visibility account-wide. See `08-knowledge/stores/policy-landmines.md` §5.

## 5. Technical factors (vitals gate visibility)

- **Android vitals bad-behavior thresholds gate visibility.** The exact numbers (user-perceived
  crash and ANR bad-behavior rates, the per-device threshold) live in their canonical home
  `08-knowledge/android/play-vitals-performance.md` — cite that, don't restate the figures here.
  What matters for ranking: exceeding a threshold makes Play **reduce the app's discovery surface**
  and may show a warning on the store listing for affected devices. Play Console flags it explicitly
  in the Vitals section — treat any such warning as a **Critical-severity ASO blocker** that must be
  cleared before metadata work.
- **App size at the install decision**: download size is shown pre-install and large APK/AAB size
  measurably cuts install completion on budget devices and metered connections — an indirect but
  real conversion (hence ranking) factor. AAB + resource shrinking first, then asset diet.
- **Target API currency**: apps below the current target-API deadline (cycles every August) lose
  availability to new users on newer Android versions — a hard distribution cut, not a soft
  ranking penalty. Baseline in `08-knowledge/android/version-matrix.md`.
- **Pre-launch report / stability on top devices**: instability concentrated on high-volume
  devices hurts more than the same rate spread thin, because per-device thresholds apply.

## 6. What does NOT work anymore (detection and penalties)

| Tactic | Why it fails | Detection | Typical penalty |
|---|---|---|---|
| Keyword stuffing (repetition, keyword lists, irrelevant terms) | Explicitly against Metadata policy | Automated text analysis; repetition thresholds | Listing rejection; rank suppression; metadata strike |
| Incentivized installs (paid/rewarded install campaigns for rank) | Velocity without retention is a recognizable signature | Cohort retention anomaly detection | Rank correction (gains erased), possible strike |
| Review exchanges / bought reviews | Against Ratings & Reviews policy | Graph analysis of reviewer accounts, velocity anomalies | Reviews purged, rating reset, strike; repeat = suspension |
| Competitor brand terms in metadata | Impersonation/IP policy | Trademark-holder complaints + automated matching | Listing rejection or takedown (see `08-knowledge/stores/policy-landmines.md` §1) |
| Title emojis / "#1", "best", "free" decoration | Metadata policy bans promotional language in title | Automated | Rejection at review |
| Rapid metadata churn to "test" keywords | Re-indexing lag means you measure noise; churn looks manipulative | — | Self-inflicted rank instability |

The common thread: Play's detection is **cohort-statistical**, not keyword-cop. Anything that
buys velocity without earning retention produces a signature that gets corrected, usually within
weeks, often with a penalty attached.

## 7. The flywheel model

```
relevance (metadata) → impressions on query
                     → conversion (icon/title/rating in results)
                     → install velocity on that keyword
                     → rank rises → more impressions
                     ↑                                  |
                     └── retention & rating confirm quality ──┘
```

Practical reading of the flywheel:

1. **Start where you can convert.** Target keywords where the app genuinely wins the comparison
   shoppers make — mid-volume, beatable competition (scoring in
   `04-aso/workflows/keyword-research.md`).
2. **Conversion assets are ranking assets.** Improving icon CTR feeds the same loop as a title
   keyword. Run both tracks together.
3. **Retention closes the loop.** If D7 is poor, ranking gains are temporary by construction.
4. **Compounding favors patience.** The flywheel accelerates slowly and decays slowly; consistent
   monthly iteration beats heroic quarterly overhauls.

## 8. Realistic timeline expectations

| Change | Indexing visible | Rank settles | Judge results after |
|---|---|---|---|
| Title / short description change | 24–72 h | 1–4 weeks | 4 weeks |
| Full description change | 24–72 h | 1–3 weeks | 3 weeks |
| Icon / screenshot change (conversion path) | Immediate display | 2–6 weeks (flywheel lag) | 4–6 weeks |
| New localization | days | 4–8 weeks | 8 weeks |
| Rating recovery campaign | — | 4–12 weeks | quarterly |

**Do not thrash.** Change one metadata variable, let it settle a minimum of 2 weeks (4 for
title), measure against pre-change baseline in Play Console search-term reports, then iterate.
Overlapping changes destroy attribution and can read as manipulative churn.

## 9. AppGallery & RuStore deltas

Both stores have **smaller indexes and simpler ranking systems** than Play — the doctrine above
applies, with these shifts (store mechanics in `04-aso/stores/huawei-appgallery.md` and
`04-aso/stores/rustore.md`):

- **Editorial weight is much higher.** Featuring, curated collections, and category placement
  drive a larger share of discovery than search. Cultivating store editorial relationships and
  applying for featuring campaigns has outsized ROI versus Play.
- **Exact-match is stronger.** Less sophisticated semantic matching means exact keyword phrases
  in the title and description matter more, and partial/synonym coverage matters less. Localized
  exact phrases (Russian for RuStore, target-market languages for AppGallery) are decisive.
- **Smaller index = lower competition.** Mid-tail keywords that are saturated on Play are often
  uncontested; a direct port of Play keyword research overweights competitiveness — redo the
  difficulty scoring per store.
- **Behavioral signals are coarser.** Per-keyword conversion feedback loops are weaker or absent;
  installs, rating magnitude, and update recency dominate the quality side.
- **RuStore**: Russian-language metadata is effectively mandatory for rank; moderation checks
  metadata claims against actual functionality strictly.
- **AppGallery**: Huawei runs frequent featuring/campaign programs by category and region;
  quality score includes AG Connect crash analytics, paralleling the vitals gate.

## 11. Paid UA × ASO interaction

Paid user acquisition and organic ASO share the same flywheel (§7), so UA changes the very signals
ASO experiments try to read. Know the interaction before running either.

- **UA feeds velocity and can lift organic rank — legitimately.** Real paid installs that *retain*
  raise install velocity and feed the per-keyword momentum signal (§3). Sustained, well-targeted UA
  to high-intent audiences produces organic rank lift as a side effect ("ASO halo"). This is the
  honest version of buying velocity: the cohort retains, so the signal holds.
- **UA changes your conversion baseline, often downward.** Paid traffic frequently converts and
  retains *worse* than organic search traffic (broader targeting, lower intent), so blending UA into
  store-listing metrics drags the listing CVR and D1/D7 numbers that ASO is judged on. Always read
  conversion and retention **split by channel** (organic search vs paid vs browse) — a listing
  experiment measured on blended traffic is measuring your media mix, not your listing.
- **Incentivized / rewarded UA backfires.** Paying for installs that don't reflect genuine demand
  (rewarded-install offerwalls, bot/farm traffic, "boost my rank" services) buys velocity without
  retention — the exact cohort-statistical signature Play corrects, with the gains erased and a
  possible strike (§6). Incentivized UA is a ranking *liability*, not a tactic.
- **UA contaminates experiment windows.** A paid burst (or a campaign starting/ending) mid-test
  shifts traffic volume and mix, invalidating store-listing A/B reads and the attribution of any
  organic metadata change (§8). Rules: freeze or hold UA spend flat across a listing-experiment
  window; never start a metadata test in the same window as a new campaign; when UA *must* run,
  segment results by channel and extend the judge-after period. Coordinate UA calendars with the
  experiment queue so windows don't collide — the experiment program (`experiment-program.md`)
  owns experiment scheduling, EXP IDs, and the channel-split read; this doc owns only the *why*.

The throughline: UA and ASO are not independent. Treat paid spend as an input to every organic
signal you measure, isolate it by channel, and keep it stable across any experiment you intend to
trust.

## 12. Cross-references

- Conversion doctrine: `08-knowledge/aso/conversion-optimization.md`
- Policy hazards that nullify ranking work: `08-knowledge/stores/policy-landmines.md`
- Keyword research procedure: `04-aso/workflows/keyword-research.md`
- Competitor analysis procedure: `04-aso/workflows/competitor-analysis.md`
- Experiment scheduling / UA-coordination: `04-aso/workflows/experiment-program.md`
- ASO audit prompt: `01-core/prompts/12-aso-audit.md`
- Launch sweep: `07-checklists/aso-launch.md`
