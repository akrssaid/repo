# RuStore — Store Operating Guide

This guide is the factory's authoritative reference for publishing and optimizing on RuStore, the
VK-operated Russian app store: why it matters, metadata and asset specs, publication requirements,
the RuStore SDK surface (push, billing, review/update APIs), the Russian-localization mandate,
legal/compliance flags, and the review process. Used by prompts `01-core/prompts/12-aso-audit.md`,
`13-keyword-research.md`, and `14-store-assets.md`.

> **Standing instruction:** store specs change. Values below are complete and current as of
> 2026-06-10, but always **verify limits in the store console (RuStore Console) before
> submission**.

---

## 1. Why RuStore matters

- **Default preinstall.** Since 2023, RuStore is mandated as a preinstalled store on smartphones
  sold in Russia — it is the path of least resistance for RU users acquiring apps.
- **Play payments are unavailable in Russia.** Google suspended Play billing for RU users in
  2022; paid apps and Play IAP do not work there. For any app monetizing RU users, **RuStore
  billing is the monetization path** — Play distribution in RU is effectively free-tier only.
- Google Play itself remains accessible in RU but with degraded commerce and an uncertain future;
  RuStore is also the fallback channel for apps removed from Play under sanctions-related
  pressure.
- Competition is thin relative to Play: solid metadata plus native-grade Russian localization can
  reach top-of-category positions that would cost serious money on Play.
- Decide investment level with `08-knowledge/stores/store-comparison.md`; if the target app has
  meaningful RU traffic in Play Console statistics, RuStore publication is a default-yes.

### 1.1 Search, featuring & editorial

- **Algorithmic search** keys on the app name (strongest), short description, and full
  description; Cyrillic terms index, so the Russian keyword research in §6 is what drives rank.
- **RuStore runs editorial collections and featuring.** The store surfaces curated collections,
  themed selections, "new & noteworthy"-style placements, and seasonal campaigns on the
  home/discovery surfaces — these are a real discovery channel on top of algorithmic search.
- How to pursue it: keep the listing complete (Russian copy, full screenshot set, video where
  available), maintain a healthy rating and low crash rate, and watch the RuStore Console /
  developer communications for collection or campaign invitations. Keep layered source art from
  `docs/factory/assets/` ready so a featuring request can be answered quickly. Editorial decisions
  favor native-grade Russian listings (§6) and stable, GMS-independent builds (§5).

## 2. Metadata fields & limits

| Field | Limit | Indexed | Notes |
|---|---|---|---|
| App name | 50 chars | Yes — strongest | Room for brand + Russian-language descriptor. Cyrillic keywords index; use the Russian term users actually search (see §6). |
| Short description | 80 chars | Yes | Same role as Play's short description; write in Russian first, not translated-from-English. |
| Full description | 4000 chars | Yes — moderate | Same discipline as Play: natural usage, primary keywords ~2–3%, no stuffing. **Verify all three limits in RuStore Console** — RuStore revises field lengths more often than Play. |
| What's new | ~500 chars | No meaningful indexing | Russian-language release notes; same update-conversion rules as other stores. |
| Category | 1 | Browse/charts | Category charts are winnable; choose deliberately. |
| Age rating (content rating) | required | Filtering | Declared via RuStore's rating questionnaire; must match content (see §4). |
| Keyword/tag input | **where present in console; verify** | — | Do not assume a dedicated keyword field exists. **Where present in the console, fill it** with top Russian keyword-map terms (`04-aso/workflows/keyword-research.md`); otherwise the name and description carry keyword weight. Verify presence at submission time. |
| Privacy policy URL | required | No | Mandatory; should be reachable from RU and ideally available in Russian. |

## 3. Graphic asset specs

| Asset | Spec | Required | Notes |
|---|---|---|---|
| App icon | 512×512 PNG | Yes | Same master as Play export (`03-design/06-icon-redesign.md`); check console for current size/weight caps at upload. |
| Screenshots | min 1, up to 10; portrait or landscape; **1080 px+ on the short side recommended** | Yes (provide 5+ in practice) | Localize screenshot captions to Russian — translated captions on the same renders is the cheap path (`04-aso/workflows/screenshot-optimization.md`). |
| Promo video | Optional (link/upload per console) | No | Provide if available; not a gating factor today. |

## 4. Publication requirements

1. **Developer verification.** Companies verify via legal-entity data; foreign individuals and
   companies can register but go through identity/entity verification — allow several business
   days. Account approval precedes any submission.
2. **Binary:** APK **or** AAB accepted. Sign with the standard release keystore
   (`10-releases/release-workflow.md`); RuStore does not require Play App Signing artifacts.
3. **Content rating** questionnaire completed honestly; RU age-labels (0+/6+/12+/16+/18+) apply.
4. **Privacy policy URL** mandatory (see §2).
5. Review notes: provide a **test account** if any flow requires login.
6. The same package name can live on Play, AppGallery, and RuStore simultaneously — keep
   `versionCode` discipline across stores per `10-releases/app-versioning-policy.md`.

## 5. RuStore SDK surface

Integration effort is real but bounded; budget **M** total (0.5–2 days) for push + billing in an
app with abstracted service layers, more if Google services are hard-wired
(`02-engineering/04-sdk-inventory.md` tells you which).

| SDK | Purpose | Notes |
|---|---|---|
| **RuStore Push SDK** | Push notifications on devices without GMS (and as an FCM alternative for RU users generally) | Server must dual-send (FCM + RuStore push) keyed by which client SDK registered; mirror of the AppGallery Push Kit pattern (`04-aso/stores/huawei-appgallery.md` §4.2). Skip if push is not a core flow. |
| **RuStore Billing (Pay) SDK** | In-app purchases & subscriptions — the only working IAP path for RU users | Separate product catalog in RuStore Console; separate server-side receipt validation endpoint; price points in RUB. Abstract billing behind an interface and ship a RuStore flavor or runtime switch. |
| **Review SDK** | In-app rating prompt (analog of Play In-App Review) | Trigger at success moments only. |
| **Update SDK / API** | In-app update prompts; publish/update automation via RuStore public API | Wire the publish API into the release workflow if updating three stores manually becomes a drag. |

The SDKs are distributed via a public Maven repository and documented at the RuStore developer
portal — verify current artifact coordinates and versions at integration time.

## 6. Localization mandate (Russian)

- A **Russian listing is required** — name/short/full description and what's-new in Russian. An
  English-only listing will underperform badly even if accepted.
- **Quality bar: native-grade, not machine-translated.** Machine output is detectable in the
  first sentence and kills conversion and reviews. Process per `04-aso/workflows/localization.md`:
  translate meaning, re-do keyword research in Russian (Russian users search Russian terms — run
  `04-aso/workflows/keyword-research.md` against RuStore/Yandex-flavored queries, do not
  transliterate English keywords), then have copy reviewed by a native speaker.
- Localize screenshot captions (§3) and respond to RU reviews in Russian.

## 7. Legal / compliance notes (flag to operator — not legal advice)

- **RU data-localization law (242-FZ):** apps collecting personal data of Russian citizens are
  required to perform initial collection/storage on servers located in Russia. This is an
  operator-level business/legal decision — **the agent must flag it in
  `docs/factory/audits/ASO_AUDIT.md` and `docs/factory/PROJECT_STATE.md` (Risks), never resolve
  it silently**. Minimizing collected personal data shrinks the exposure.
- Sanctions/counter-sanctions context affects payouts: receiving RuStore monetization revenue to
  non-RU bank accounts has practical constraints — verify the current payout mechanics in the
  RuStore Console and with the operator's accountant before turning on paid features.
- Content restrictions under RU law (age labels, prohibited-content categories) are enforced at
  review; the content-rating questionnaire (§4) is the main surface.
- **Privacy / data-collection declaration.** RuStore's equivalent of Play's Data safety form is
  driven by the same source of truth: the SDK inventory. Map each SDK's data collection/sharing
  and complete RuStore's privacy declaration from it — follow
  `04-aso/workflows/data-safety-listing.md` (it covers the per-store privacy-declaration
  procedure, RuStore included) rather than declaring ad hoc. Keep the declaration, the privacy
  policy URL (§2), and the actual SDK behavior consistent.

## 8. Review process & typical timelines

- Moderation is human + automated; typical turnaround **1–3 business days**, often under 24h for
  updates to an already-published app. First submission and account verification take longest.
- Rejections arrive with a reason in the console; fix-and-resubmit cycles are fast.
- No staged-rollout mechanism comparable to Play — treat each release as 100% rollout and gate it
  with `01-core/prompts/15-final-verification.md` accordingly.

## 9. Common rejection causes

| Rejection cause | Prevention |
|---|---|
| Privacy policy URL missing or unreachable | §2; test the URL from an RU-reachable network. |
| No test account for login-gated functionality | Always attach demo credentials in review notes. |
| Russian listing missing or visibly machine-translated | §6 quality bar before submission. |
| App crashes or core feature requires GMS on a GMS-free device | Same detection procedure as AppGallery (`04-aso/stores/huawei-appgallery.md` §4.1). |
| Content rating mismatch with actual content | Re-answer the questionnaire after any content-affecting feature change. |
| Screenshots/description misrepresent the app | Generate assets from the actual build (`01-core/prompts/11-screenshot-generation.md`). |
| Trademark/brand conflicts in name or assets | Search RuStore for conflicts; keep brand-ownership proof. |
| Developer verification incomplete or documents inconsistent | Finish account verification before scheduling the release date. |

## 10. Pre-submission checklist hook

Before any RuStore submission: §2 limits re-verified in RuStore Console, Russian copy at native
grade (§6), assets per §3, test account attached, data-localization flag raised to operator if
personal data is collected (§7), and the listing logged in `docs/factory/CHANGELOG.md`. Then run
`07-checklists/aso-launch.md`.
