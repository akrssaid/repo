# Store Listing Conversion Optimization

Evergreen doctrine for converting store impressions into installs. Agents consult this when
producing or auditing icons, screenshots, descriptions, and rating strategy during P6–P7. This is
the "what and why"; execution procedures live in `04-aso/workflows/screenshot-optimization.md`,
`04-aso/workflows/icon-optimization.md`, and `01-core/prompts/14-store-assets.md`. Conversion and
ranking are one system — see `08-knowledge/aso/ranking-factors.md` §3 for why conversion rate per
query is itself a ranking input.

**Last reviewed: 2026-06-10.** Store UI layouts, A/B tooling (Play "store listing experiments"),
and benchmark ranges drift — verify against the current Play Console UI and official docs before
quoting limits or designing experiments.

---

## 1. The funnel and where each asset acts

```
search impression → (tap) → listing view → (install) → install → (keep) → retained user
```

| Stage | What the user sees | Assets doing the work |
|---|---|---|
| Search results | Icon + title + rating (+ download count) | Icon, first 30 title chars, rating display |
| Listing, first screen | Screenshots 1–3 (gallery dominates), short description, rating block | Screenshot 1 above all; icon repeats |
| Listing, scrolled | Remaining screenshots, video, description fold, reviews | Screenshots 4–8, top visible reviews |
| Post-install | First-run experience | Outside ASO scope but closes the loop (retention → rank) |

Two distinct battles:

- **Search-results CTR** is won by the icon + title + rating trio alone. Screenshots are not
  visible there for most placements. If impressions are high and listing views are low, fix the
  trio — not the screenshots.
- **Listing-page conversion** is won by screenshots. Most visitors decide from the gallery
  without reading anything. If listing views are high and installs are low, fix screenshot 1
  first.

Diagnose which battle you're losing from Play Console → store performance (impressions → listing
views → installs breakdown) before touching any asset.

## 2. The 3-second rule and first-screenshot primacy

A store visitor gives the listing roughly three seconds before the keep/bounce decision. In that
window they see: icon, title, rating, and screenshot 1 (partially 2). Consequences:

1. **Screenshot 1 is the single highest-leverage listing asset.** It must communicate the core
   benefit *as a caption-led marketing frame*, readable at thumbnail size, without requiring the
   user to understand the UI.
2. **Never lead with a settings screen, empty state, splash, or login.** Lead with the app
   mid-success: the document open, the file recovered, the video downloaded.
3. **Caption first, UI second.** A 3–6 word benefit headline at the top of the frame carries the
   message; the UI screenshot beneath provides proof.
4. **Portrait screenshots** show 2–3 abreast in the gallery; the first two frames can be designed
   as a connected panorama, but each must also survive standing alone.

## 3. Asset impact ranking

Empirical priority order for a utility app (spend effort top-down):

```
icon ≈ screenshot 1  >  rating display  >  screenshots 2–3  >  video  >  short desc  >  full desc
```

- **Icon**: acts at every surface (search, listing, home screen, updates). One concept, high
  silhouette contrast, no words, distinct from the top 5 competitors on your primary keyword.
  Principles in `08-knowledge/design/app-icon-principles.md`; workflow in
  `04-aso/workflows/icon-optimization.md`.
- **Rating display**: not directly an "asset" you draw, but a 4.4 vs 3.9 difference moves CTR
  more than any screenshot polish. Rating management (§6) is conversion work.
- **Video**: optional for utility apps; poorly made video *reduces* conversion (it replaces the
  gallery's first slot with a play button on some layouts). Add only when a polished 15–30 s
  capture exists. Screenshots first, always.
- **Description**: almost nobody reads the full description — **but the ranking algorithm does**
  (see `08-knowledge/aso/ranking-factors.md` §2). Write the full description for the index and
  the first three lines for humans (§7).

## 4. Screenshot messaging frameworks

Pick one framework per listing and execute it consistently (full procedure in
`04-aso/workflows/screenshot-optimization.md`):

1. **Benefit-led captions** (default for utilities): each frame = one benefit headline ("Recover
   deleted photos in seconds") + UI proof. Order frames by benefit importance discovered in
   keyword research, not by app navigation order.
2. **Problem → solution sequence**: frame 1 names the pain ("Phone storage full?"), frames 2–4
   show the resolution. Strong for cleaner/recovery/file-manager categories; risky if the
   "problem" framing drifts into fear-mongering — see deceptive-claims rules in
   `08-knowledge/stores/policy-landmines.md` §1.
3. **Social-proof embedding**: one mid-gallery frame carrying a real review quote or aggregate
   ("Trusted by 5M+ users — 4.6★") — only with verifiable numbers. Fabricated counts or awards
   are a metadata-policy violation.

Universal rules: caption text ≥ 1/12 of frame height (thumbnail-legible); device frames optional
but consistent; localize captions for every shipped locale (`04-aso/workflows/localization.md`);
keep one visual system (colors from the app's design tokens) across all frames.

## 5. Description conversion structure

Only the first ~3 lines (~250 chars) render before "Read more" — this is the only part humans
read. Hook formula for those lines:

```
[Core benefit in user words] + [credibility marker] + [differentiator]
"Recover deleted photos, videos and documents — fast, free, no root.
Trusted by 5M+ users. Works even without a backup."
```

Body (read by the algorithm, skimmed by maybe 2% of humans): short benefit-led paragraphs, a
feature list with ✓/– bullets for skimmability, priority keywords appearing naturally per the
canonical density rule (`08-knowledge/aso/ranking-factors.md` §2 — 2–4 natural uses of a primary
keyword per 4000 chars, hard cap 5), and a closing call-to-action. Short description (80 chars)
doubles as both an indexed
field and visible copy on some surfaces — make it a complete benefit sentence containing the #1
keyword, not a keyword list.

## 6. Rating management

- **In-app review API timing**: trigger `ReviewManager` only after a *success moment* — file
  recovered, document saved, task completed — **never on app launch, never mid-task, never after
  an error or an ad**. Wait at least 2–3 sessions and one completed core action before the first
  ask. The API is quota-limited and may silently not show; never build UI that promises a dialog.
- **Rating-gate ethics and the policy line**: pre-filtering ("enjoying the app?" → happy users to
  Play, unhappy users to a feedback form) is widespread. Google's policy bars *incentivizing* or
  *manipulating* reviews; a neutral feedback-first branch is currently tolerated, but **gating
  must never block, discourage, or add friction to the option to review** and you must not ask
  only users you've classified as happy via the official API path. When in doubt take the
  conservative reading — verify against the current Ratings & Reviews policy text.
- **Reply doctrine**: reply to every 1–2★ review within 72 h — fix-or-explain, never argue;
  users can revise ratings after replies and a visible reply pattern lifts conversion for review
  readers. Reply to a sample of 5★ reviews to reinforce keywords organically appearing there.
- **Recency beats history**: Play weights recent ratings; a bad legacy average can be recovered
  in one or two quarters of disciplined success-moment prompting after a quality release.

## 7. A/B testing discipline

Play Console store listing experiments (icon, screenshots, descriptions; verify current
capabilities — the toolset changes):

1. **One variable per experiment.** Icon OR screenshot set OR description — never combined, or
   the result is unattributable.
2. **Full weeks, minimum one, prefer two.** Weekday/weekend behavior differs; never stop an
   experiment mid-week or at "looks significant" after 2 days.
3. **Seasonality awareness**: don't run experiments across holidays, major sales periods, or a
   featuring spike; the cohort isn't representative.
4. **Minimum traffic**: below ~1,000 listing visitors per arm per week, results are noise —
   prefer sequential before/after measurement with 4-week windows instead.
5. **Record everything** in the target repo: hypothesis, variant, dates, result, decision —
   in `docs/factory/audits/ASO_AUDIT.md` follow-up log per `09-templates/aso-audit-report.md`.
6. Ship winners fully, then wait 2 weeks before the next experiment (ranking settle time,
   `08-knowledge/aso/ranking-factors.md` §8).

## 8. Conversion benchmarks (estimates — calibrate per app)

All figures are **rough planning estimates** for utility-category apps on Google Play, blended
across geos; actual values vary by keyword intent, brand strength, and geo mix. Use Play
Console's built-in peer comparison for your real category benchmark.

| Metric | Weak | Typical | Strong |
|---|---|---|---|
| Search results CTR (impression → listing view or direct install tap) | <2% | 3–5% | >7% |
| Listing view → install (browse/search blended) | <15% | 20–30% | >35% |
| Search-driven listing conversion (high-intent queries) | <25% | 30–40% | >50% |
| First-week install → D7 retained | <8% | 12–20% | >25% |

Reading the table: high-intent search traffic ("pdf reader") converts far above browse traffic;
a falling blended conversion rate after a featuring event is usually mix-shift, not asset decay.
Always segment by acquisition channel before reacting.

## 9. Cross-references

- Ranking interplay: `08-knowledge/aso/ranking-factors.md`
- Policy limits on claims and review practices: `08-knowledge/stores/policy-landmines.md`
- Icon design principles: `08-knowledge/design/app-icon-principles.md`
- Screenshot workflow: `04-aso/workflows/screenshot-optimization.md`
- Store assets prompt: `01-core/prompts/14-store-assets.md`
- Listing template: `09-templates/store-listing.md`
- Pre-launch sweep: `07-checklists/aso-launch.md`
