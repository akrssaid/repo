# 04-aso — App Store Optimization System

This directory is the factory's complete ASO operating system: per-store operating guides and
reusable workflows that Claude executes during **P7 — ASO & Store Assets** (and revisits at every
release). It tells the agent how each store indexes, ranks, converts, and rejects — so listings are
built right the first time and improved with evidence, not guesses.

**The ASO lifecycle does not end at launch — it extends PAST it.** Once an app is live, ASO becomes
a standing monitoring loop: watch KPIs, mine and respond to reviews, run listing experiments, and
feed wins back into the listing. The post-launch workflows below (`ratings-reviews.md`,
`experiment-program.md`, `post-launch-monitoring.md`) and the post-release checklist
`07-checklists/aso-post-release.md` govern that loop; KPI snapshots and experiment outcomes land in
`docs/factory/PROJECT_STATE.md` and `docs/factory/reports/EXPERIMENT_LOG.md`.

Scales, naming, branch/commit/tag formats, and all shared conventions referenced in this directory
are defined once in `01-core/CONVENTIONS.md` — cite it; do not restate variant definitions here.

> **Standing instruction (applies to every document in this directory):** store specifications
> change without notice. All limits, dimensions, and policy thresholds below are complete and
> current as of 2026-06-10, but you must **verify limits in the store console before submission**.

---

## Stores covered

| Store | Guide | Why it is in scope |
|---|---|---|
| Google Play | `04-aso/stores/google-play.md` | Primary global distribution; richest ASO toolset (experiments, custom listings); strictest automated policy enforcement. |
| Huawei AppGallery | `04-aso/stores/huawei-appgallery.md` | Reaches GMS-free Huawei/Honor devices; editorial featuring is a real growth channel; requires HMS dependency analysis. |
| RuStore | `04-aso/stores/rustore.md` | VK-operated store preinstalled on devices sold in Russia; Google Play payments are unavailable in RU, so RuStore billing is the RU monetization path. |

Cross-store policy traps and a side-by-side comparison live in
`08-knowledge/stores/policy-landmines.md` and `08-knowledge/stores/store-comparison.md`.

## Workflow documents

| Workflow | File | Produces | Consumed by |
|---|---|---|---|
| Keyword research | `04-aso/workflows/keyword-research.md` | Keyword map (`09-templates/keyword-map.md` instance) per store, per locale. | Prompt 13 (P7) |
| Competitor analysis | `04-aso/workflows/competitor-analysis.md` | Competitor matrix feeding keyword and asset decisions. | Prompts 12–13 (P7) |
| Screenshot optimization | `04-aso/workflows/screenshot-optimization.md` | Conversion-ordered screenshot sets per device class and locale. | Prompts 11, 14 (P7) |
| Icon optimization | `04-aso/workflows/icon-optimization.md` | Icon variants and test plan (pairs with `03-design/06-icon-redesign.md`). | Prompts 10, 14 (P7) |
| Localization | `04-aso/workflows/localization.md` | Localized listings; priority locale list; native-grade quality bar. | Prompt 14 (P7) |
| Data-safety listing | `04-aso/workflows/data-safety-listing.md` | Data safety / privacy declaration procedure mapping SDK collection to each store's form. | Prompt 14 (P7); store guides §Data safety |
| Ratings & reviews | `04-aso/workflows/ratings-reviews.md` | Review-mining, response playbook, in-app review-prompt policy. | Post-launch loop; `07-checklists/aso-post-release.md` |
| Experiment program | `04-aso/workflows/experiment-program.md` | Store-listing experiment design + `EXP-NNN` queue; logs to `reports/EXPERIMENT_LOG.md`. | Post-launch loop; ASO_HANDOVER EXP-queue rows |
| Post-launch monitoring | `04-aso/workflows/post-launch-monitoring.md` | Standing KPI-monitoring cadence; snapshots into PROJECT_STATE; triggers re-optimization. | Post-launch loop; `07-checklists/aso-post-release.md` |

The first five workflows run **during P7** to build the listing. The last three run **after launch**
as the standing monitoring loop described above — they are the post-release half of the ASO
lifecycle, gated by `07-checklists/aso-post-release.md`.

## Which prompts consume this directory

| Prompt | Reads | Writes (in target repo) |
|---|---|---|
| `01-core/prompts/12-aso-audit.md` | All three store guides; `08-knowledge/aso/ranking-factors.md`, `08-knowledge/aso/conversion-optimization.md` | `docs/factory/audits/ASO_AUDIT.md` |
| `01-core/prompts/13-keyword-research.md` | `workflows/keyword-research.md`, `workflows/competitor-analysis.md`, store guides (index rules per store) | Keyword map under `docs/factory/reports/` |
| `01-core/prompts/14-store-assets.md` | Store guides (asset spec tables), `workflows/screenshot-optimization.md`, `workflows/icon-optimization.md`, `workflows/localization.md`, `09-templates/store-listing.md` | Listing copy + assets under `docs/factory/assets/` |

Output flows into `05-handover/ASO_HANDOVER_SCHEMA.md`-conformant handovers and is gated by
`07-checklists/aso-launch.md` before release (P8, `01-core/prompts/16-release-preparation.md`).
After release, the post-launch workflows are gated by `07-checklists/aso-post-release.md` and feed
KPI snapshots into `docs/factory/PROJECT_STATE.md` and experiment outcomes into
`docs/factory/reports/EXPERIMENT_LOG.md`. ID prefixes (ASO, KW, EXP) follow the registry in
`01-core/CONVENTIONS.md`.

## ASO doctrine (non-negotiable)

1. **Relevance before volume.** Rank #5 for a keyword that describes what the app actually does
   beats rank #50 for a high-volume keyword it half-matches. Irrelevant traffic converts poorly
   and degrades the retention signals every store now weighs.
2. **Conversion assets matter as much as keywords.** Icon, screenshots, and the first sentence of
   the description decide whether an impression becomes an install. A keyword win with a weak icon
   is wasted; treat asset work (`workflows/screenshot-optimization.md`,
   `workflows/icon-optimization.md`) as half the ASO budget.
3. **Never violate policy for rank.** No keyword stuffing, no competitor brand names in metadata,
   no incentivized installs or reviews, no misleading screenshots. A suspension costs more than
   any ranking gain. When in doubt, check `08-knowledge/stores/policy-landmines.md` and choose the
   conservative option.
4. **Localization is the cheapest growth lever.** A native-grade localized listing in a mid-size
   market routinely outperforms months of keyword tuning in a saturated English market. Run
   `workflows/localization.md` before paying for any acquisition.

## Operating rules for agents

1. Read the relevant store guide **in full** before touching any listing field.
2. Every metadata change is recorded in `docs/factory/CHANGELOG.md` with date, store, field,
   old value, new value, and rationale — ASO is an experiment log, not a one-shot task.
3. One variable at a time where the store allows experiments (Google Play); for stores without
   experiments (AppGallery, RuStore), change metadata on release boundaries and annotate the date
   so install-curve shifts are attributable.
4. Asset specs in these guides are authoritative for generation, but re-verify in the console at
   upload time (see standing instruction above).
5. After launch, run the monitoring loop (`workflows/post-launch-monitoring.md`,
   `workflows/ratings-reviews.md`, `workflows/experiment-program.md`) on the cadence in
   `07-checklists/aso-post-release.md`. Severity/effort/status labels and ID formats come from
   `01-core/CONVENTIONS.md` — do not invent local scales.

## Related knowledge

- `01-core/CONVENTIONS.md` — scales, ID registry (ASO/KW/EXP), naming, and shared conventions (cited, not restated, throughout this directory).
- `08-knowledge/aso/ranking-factors.md` — deep dive on what each algorithm weighs; canonical keyword-density rule.
- `08-knowledge/aso/conversion-optimization.md` — listing CVR playbook.
- `08-knowledge/stores/store-comparison.md` — when to invest in which store.
- `08-knowledge/stores/data-safety-mapping.md` — SDK-to-declaration mapping behind `workflows/data-safety-listing.md`.
- `02-engineering/04-sdk-inventory.md` — feeds Data safety (Play) and HMS analysis (AppGallery).
- `07-checklists/aso-post-release.md` — gates the post-launch monitoring loop.
