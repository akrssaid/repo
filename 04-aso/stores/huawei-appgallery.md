# Huawei AppGallery — Store Operating Guide

This guide is the factory's authoritative reference for publishing and optimizing on Huawei
AppGallery: metadata fields as named in AppGallery Connect, asset specs, the HMS-vs-GMS reality
check (the engineering question that decides whether publishing is even viable), search and
featuring behavior, review-process quirks, and payout/account notes. Used by prompts
`01-core/prompts/12-aso-audit.md`, `13-keyword-research.md`, and `14-store-assets.md`.

> **Standing instruction:** store specs change. Values below are complete and current as of
> 2026-06-10, but always **verify limits in the store console (AppGallery Connect) before
> submission**.

---

## 1. Why AppGallery is in scope

The GMS-free audience is narrower than "all Huawei/Honor" — scope the claim precisely:

- **Huawei-branded devices** shipped since the May 2019 US-export restriction lack Google Mobile
  Services (GMS): no Play Store, no Play billing, no FCM push, no Google Maps/Sign-In.
- **Honor devices from BEFORE the November 2020 sale** (when Huawei sold Honor) are likewise
  GMS-free. **Honor devices launched AFTER the 2020 split ship GMS** — they have the Play Store and
  are NOT the AppGallery-only audience. Do not assume modern Honor hardware is GMS-free.

AppGallery is the default store on the GMS-free devices above — a large, under-competed audience,
strongest in China, MEA, LATAM, and parts of Europe. Competition per keyword is far lower than
Play, and editorial featuring is genuinely attainable for a solo publisher. The catch is
engineering: see §4 before committing.

## 2. Metadata fields & limits (AppGallery Connect names)

| AppGallery Connect field | Limit | Indexed | Notes |
|---|---|---|---|
| **App name** | 64 chars | Yes — strongest | More room than Play's 30; use it for brand + descriptor + one secondary keyword, still readable. Per-language values supported. |
| **Brief introduction** (a.k.a. brief intro) | 80 chars | Yes | Equivalent of Play's short description; shown prominently on the listing card. |
| **App description** (a.k.a. introduction) | 8000 chars | Yes — moderate | Twice Play's length. Same discipline: natural sentences, primary keywords ~2–3% density, no stuffing — Huawei's human reviewers reject stuffed copy. Reuse the Play description and extend with feature detail and FAQ-style sections rather than padding. |
| **What's new** (New features) | 500 chars | No meaningful indexing | Update-conversion text; same rules as Play's what's-new. |
| App category + subcategory | 1 each | Browse/featuring | Category choice affects which editorial team sees the app. |
| Privacy policy URL | required | No | Mandatory; must be reachable from review networks (avoid geo-blocking China/EU reviewer IPs). |
| Search keywords field | **where present in console; verify** | — | Do not assume a dedicated keyword/tag input exists. AppGallery Connect exposes one only for some accounts/regions and changes this over time. **Where present in the console, fill it** with the top keyword-map terms (`04-aso/workflows/keyword-research.md`); otherwise place those terms in the app name and description. Verify presence in the console at submission time. |

## 3. Graphic asset specs

> **This section is the AUTHORITATIVE source for AppGallery screenshot counts.** Prompt 11
> (`01-core/prompts/11-screenshot-generation.md`) cites the **3–10 per device type** range from
> here; it must not restate a different count.

| Asset | Spec | Required | Notes |
|---|---|---|---|
| App icon | **216×216 PNG**, ≤ 2 MB, no rounded-corner baked in | Yes | Smaller canvas than Play's 512 — re-export from the icon master (`03-design/06-icon-redesign.md`), do not upscale or just downscale the Play PNG without checking legibility at 216 px. |
| Screenshots | **3–10 per device type**; portrait 1080×1920 (or other 9:16 such as 1440×2560), landscape 16:9 accepted; JPG/PNG ≤ 2 MB each | Yes (min 3) | Per device type: phone required; tablet set strongly recommended if the app supports tablets. Allowed resolutions are enumerated in the console — verify before export. |
| Promo video | Optional; uploaded file or link per console rules (not YouTube — YouTube is blocked in China; the console accepts direct upload) | No | Provide for featuring candidacy; editorial teams prefer apps with complete media. |
| Featuring/banner art | Requested by editorial team when featuring is offered | No | Keep layered source files from `docs/factory/assets/` ready so a featuring offer can be answered within days. |

## 4. HMS reality check (do this before anything else)

AppGallery devices have **no GMS**. If the app hard-depends on Google services, it will install
but break. Run this analysis during P7 using `02-engineering/04-sdk-inventory.md` output.

### 4.1 GMS hard-dependency detection procedure

1. From the SDK inventory, list every `com.google.android.gms.*`, `com.google.firebase.*`,
   `com.android.billingclient.*` dependency.
2. Confirm at the artifact level:
   ```powershell
   # In the target app repo:
   .\gradlew :app:dependencies --configuration releaseRuntimeClasspath | Select-String "com.google.android.gms|com.google.firebase|billingclient"
   ```
3. Classify each hit: **hard** (feature unusable without it — e.g. Maps SDK rendering the main
   screen, Google Sign-In as the only auth, FCM-driven core flows, Play Billing as the only
   payment path, AdMob as the only ad stack) vs **soft** (analytics, optional sign-in, crash
   reporting — degrades silently).
4. Runtime check on a GMS-free device/emulator (Huawei cloud-debugging in AppGallery Connect
   works): exercise every core flow; log crashes/`ApiException`s.
5. Record findings in `docs/factory/audits/ASO_AUDIT.md` under "AppGallery viability".

### 4.2 HMS Core counterparts

| Google dependency | HMS Core counterpart | Port effort (typical) |
|---|---|---|
| Google Maps SDK | **Map Kit** | M–L — API-similar but not drop-in; map styling and clustering differ. |
| Google Sign-In / Identity | **Account Kit** | S–M — straightforward if auth is abstracted. |
| Firebase Cloud Messaging | **Push Kit** | M — server side must dual-send (FCM + Push Kit) keyed by device capability. |
| Play Billing | **IAP (In-App Purchases Kit)** | M–L — separate product catalog in AppGallery Connect, separate receipt validation. |
| AdMob / Google Ads | **Ads Kit (Petal Ads)** | S–M client-side; revenue/fill varies by region — validate eCPM before committing. |
| Firebase Analytics / Crashlytics | **Analytics Kit / AGConnect Crash** | S. |

Pattern: isolate Google services behind interfaces, ship an `hms` product flavor. Detection at
runtime via availability checks (GMS `GoogleApiAvailability` vs HMS `HuaweiApiAvailability`).

### 4.3 Graceful-degradation alternative

When the HMS port is not worth the effort (small RU/MEA/CN revenue expectation, heavy GMS
coupling): ship the existing APK/AAB **only if** every hard dependency is feature-flagged off or
replaced with a neutral fallback (web map view, email auth, in-app polling instead of push, no
ads). Hide—don't break—features on GMS-free devices. If core value cannot survive degradation,
**do not publish to AppGallery**; a 1-star-magnet listing is worse than absence. Decision and
rationale go in `docs/factory/PROJECT_STATE.md`.

## 5. Search & featuring behavior

- Algorithmic search exists (app name strongest, then brief intro/description) but the index is
  less sophisticated than Play's and search volume per keyword is lower. Exact-match terms in the
  app name carry disproportionate weight.
- **Editorial featuring matters more than algorithmic search** — the inverse of Play. Featured
  placements, themed collections, and seasonal campaigns drive the bulk of discovery installs.
- How to pursue featuring:
  1. Complete listing: all locales relevant to target regions, video, tablet screenshots.
  2. Integrate at least the low-effort HMS kits (Analytics, Push) — editorial teams favor
     HMS-integrated apps.
  3. Apply via AppGallery Connect promotion/featuring forms and regional operation team contacts
     (invitations to campaigns appear in the console and via account-manager email once the app
     has traction).
  4. Time applications to Huawei campaign seasons (new-device launches, regional holidays).
- Quality scores (crash rate, rating, completeness of listing) gate featuring eligibility.

## 6. Review process quirks & account notes

- **Stricter, more manual content review than Play.** Human reviewers exercise the app; typical
  turnaround 1–3 business days, longer on first submission or after rejection.
- **Copyright/qualification documents:** some categories and regions (notably games anywhere,
  finance, health, news, VPN; almost everything for China distribution) require qualification
  documents — copyright certificates, licenses, business credentials.
- **China vs global split:** China distribution requires a Chinese business entity, ICP filing,
  and game licenses where applicable. **Global developers target "global" distribution only** —
  deselect China in the release-country list; do not waste cycles on China requirements.
- Account: individual and enterprise developer accounts exist; identity verification (passport /
  business documents) is required and can take several days — start account setup early.
- Payout: AppGallery Connect supports settlement to international bank accounts; IAP revenue
  share is broadly comparable to other stores (verify current terms in the console). Minimum
  payout thresholds and monthly settlement cycles apply.

## 7. Common rejection causes

| Rejection cause | Prevention |
|---|---|
| App crashes or core feature broken on GMS-free review device | §4 procedure; test on real/cloud Huawei device before submitting. |
| Privacy policy missing, unreachable from reviewer region, or not matching actual collection | Host on a globally reachable URL; reconcile with SDK inventory. |
| Permissions requested without in-app explanation | Reviewers check that each runtime permission has a visible purpose; align with `02-engineering/06-manifest-audit.md`. |
| Missing qualification/copyright documents for the category | Check category requirements in AppGallery Connect before choosing category (§6). |
| Metadata mismatch: screenshots or description show features absent in the build | Regenerate assets from the actual build (`01-core/prompts/11-screenshot-generation.md`). |
| Keyword-stuffed name or description | §2 discipline; reviewers reject stuffed copy manually. |
| Login required but no test account provided | Supply demo credentials in the review-notes field on every submission. |
| App name conflicts with an existing trademark/app | Search AppGallery for the name first; keep proof of brand ownership. |

## 8. Pre-submission checklist hook

Before any AppGallery submission: §4 viability verdict recorded, all §2 limits re-verified in
AppGallery Connect, §3 assets exported at exact specs (icon at 216×216 checked for legibility),
test account in review notes, China deselected (global developers), and the listing logged in
`docs/factory/CHANGELOG.md`. Then run `07-checklists/aso-launch.md`.
