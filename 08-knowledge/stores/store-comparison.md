# Store Comparison — Google Play / AppGallery / RuStore

Strategic reference for deciding where to publish, in what order, and with what engineering
investment. Agents consult this at P1 (discovery — which stores does/should the target ship to?),
P4 (modernization — flavor and billing architecture), and P7–P8 (per-store listings and release
sequencing). Per-store operational mechanics live in `04-aso/stores/google-play.md`,
`04-aso/stores/huawei-appgallery.md`, and `04-aso/stores/rustore.md`.

**Last reviewed: 2026-06-10.** Store fees, payout rails, SDK availability, and review SLAs change
frequently — figures below are planning-grade; verify against current official store documentation
before committing to a port or quoting economics.

---

## 1. Comparison table

| Dimension | Google Play | Huawei AppGallery | RuStore |
|---|---|---|---|
| Reach | ~2.5B+ active devices globally (ex-China GMS base) | Hundreds of millions, concentrated on Huawei/Honor-legacy devices; strong in China (separate regime), MEA, LATAM pockets, parts of EU | Russia-focused; default-installed on devices sold in RF; the primary distribution channel for RU users since Play payment cutoff |
| Device base | All GMS Android | Huawei devices, many without GMS (HMS-only since 2020 hardware) | Standard GMS-era Android devices in Russia (GMS apps run; Play Billing doesn't for RU accounts) |
| Review strictness / speed | Heavily automated; hours–7 days; strictness concentrated in permissions, data safety, sensitive categories | Human-heavy; slower (days); strict on copyright documentation and claim-vs-function match | Human moderation; typically days; strict on RF-law content rules and metadata accuracy; Russian listing effectively required |
| Monetization rails | Play Billing (mandatory for in-app digital goods); service fee 15% to first $1M/yr then 30%, 15% for subscriptions (verify current terms) | HMS IAP (In-App Purchases Kit); regional payout support; commission broadly comparable — verify current AG terms | RuStore billing (pay SDK) with RF-domestic payment methods (cards/MIR, SBP, mobile); the only practical paid rail for RU users |
| Ads SDK compatibility | AdMob (requires GMS), AppLovin MAX, all major networks | **AdMob non-functional on GMS-less devices** — HMS Ads Kit, or GMS-independent networks (AppLovin, Yandex Ads, ironSource-class) | AdMob technically loads on GMS devices but RU ad demand/payouts are blocked — **Yandex Ads is the primary monetizing network**, AppLovin partial |
| Push | FCM (GMS) | HMS Push Kit (FCM dead on GMS-less hardware) | RuStore Push (VK-infrastructure) where FCM delivery to RU is degraded; FCM still works on GMS devices but reliability/policy risk noted |
| Update mechanics | Staged rollouts, in-app updates API, automatic updates | AG staged release, AG-specific in-app update API via AppGallery Connect | RuStore in-app update SDK; auto-update via RuStore app |
| Developer fees / payout | $25 one-time registration; payouts via Google Payments to supported countries | Free registration; payouts via Huawei developer settlement | Free registration; payouts in RUB to RF accounts (non-RF developer payout paths exist — verify current state) |

## 2. Decision framework — when is each store worth the port cost?

**Google Play** is non-optional: it is the canonical store, the ranking/review baseline, and the
hardest gatekeeper — passing Play review first de-risks the others.

**RuStore is worth it when** any of:
- Analytics show a meaningful RU user base (check Play Console geo stats at P1 discovery).
- The app monetizes via IAP/subscriptions and has RU users — **Play Billing is unavailable to
  Russian users**, so RU revenue is literally unrecoverable without RuStore (or alternative) billing.
- The category has strong RU demand (utilities, document tools, downloaders index well there).
Port cost is low for ads-only apps (swap to Yandex Ads in a flavor) and moderate for IAP apps
(billing abstraction needed, §3).

**AppGallery is worth it when** any of:
- The category shows Huawei-base demand (file tools, readers, system utilities do).
- A featuring opportunity exists — AG editorial featuring delivers install spikes Play rarely
  gives small developers (see `08-knowledge/aso/ranking-factors.md` §9 on editorial weight).
- The app already avoids hard GMS dependencies, making the HMS port cheap.
Port cost is driven almost entirely by GMS-coupling: an app using FCM + AdMob + Play Billing +
Maps needs four HMS substitutions; a self-contained offline utility needs nearly none.

**Skip a store when**: the app's core depends on a service unavailable there (e.g., a
YouTube-adjacent feature already borderline on Play is worse elsewhere — see
`08-knowledge/stores/policy-landmines.md` §3), or projected store revenue < one release cycle of
maintenance cost. Record the decision and revisit date in `docs/factory/PROJECT_STATE.md`.

**Port-cost estimator** (use factory effort scale S/M/L/XL):

| App profile | AppGallery port | RuStore port |
|---|---|---|
| Offline utility, ads-only | S (swap ads SDK in flavor) | S (Yandex Ads flavor) |
| Ads + remove-ads IAP | M (HMS IAP for one SKU) | M (RuStore pay for one SKU) |
| Subscriptions + push + analytics | L (IAP + Push Kit + AG Connect) | M–L (pay SDK + push + webhook ingestion) |
| GMS-coupled (Maps, ML Kit, Drive sync) | XL (per-service HMS substitution or feature cut) | M (GMS runs on RU devices; only billing/ads/push swap) |

Key asymmetry to remember: **RuStore targets GMS devices** — the port is a monetization-rail
swap, not a platform swap. **AppGallery targets GMS-less devices** — the port is a platform
substitution exercise whose cost scales with every Google service the app touches.

## 3. Multi-store engineering strategy

**Flavor-per-store (recommended default)** — Gradle product flavors `gplay` / `appgallery` /
`rustore` in a `store` dimension:

```kotlin
// build.gradle.kts (target app)
android {
    flavorDimensions += "store"
    productFlavors {
        create("gplay")      { dimension = "store" }
        create("appgallery") { dimension = "store" }
        create("rustore")    { dimension = "store" }
    }
}
```

- Store-specific SDKs (billing, push, ads) live only in flavor source sets and flavor-scoped
  dependencies — this keeps Play builds free of unused SDKs (a Data-safety and APK-size win) and
  AG builds free of GMS stubs.
- **Runtime detection** (one binary, detect GMS/HMS availability at runtime) is acceptable only
  for read-only capabilities (push token acquisition fallback). It is wrong for billing — every
  store **requires its own billing rail and forbids/penalizes shipping competitors' billing**
  in the same binary, and bundling all SDKs bloats size for everyone.
- **Billing abstraction layer**: define one internal interface (`BillingGateway`: querySkus,
  launchPurchase, acknowledge, restore, observeEntitlements) with three flavor implementations
  (Play Billing / HMS IAP / RuStore pay SDK). Entitlement state is stored once, locally
  (DataStore), fed by whichever rail is compiled in. Same pattern for push
  (`PushTokenProvider`) and ads (`AdsFacade` per `08-knowledge/monetization/ads-patterns.md`).
- **versionCode strategy per store**: keep one source-of-truth versionName, and reserve
  versionCode ranges or offsets per store so a hotfix to one store never collides with another —
  the authoritative scheme is defined in `10-releases/app-versioning-policy.md`; follow it, do
  not improvise per app.

## 4. Multi-store release sequencing

1. **Play first, as canary.** Strictest automated review, best vitals/rollout tooling. Release
   via staged rollout (10% → 50% → 100%) per `10-releases/release-workflow.md`.
2. **Hold ports until the Play rollout reaches 100%** with vitals clean for ≥3–7 days — crash
   fixes found at Play 10% are free for AG/RuStore users.
3. **Then submit AppGallery and RuStore in parallel** (their human reviews run days; pipelining
   saves a week per cycle).
4. **Expect version skew** and design for it: server APIs and remote configs must tolerate the
   ports running one release behind Play during the gap.
5. Hotfix exception: a security/crash hotfix ships to all stores simultaneously, skipping the
   canary hold.
6. **Track per-store release state** in the target's `docs/factory/PROJECT_STATE.md` (current
   versionCode per store, submission date, review status) and log every store submission in
   `docs/factory/CHANGELOG.md` — multi-store version skew is unmanageable from memory.
7. Review-time budgeting for release planning: Play hours–7 days (automated-path typical <24 h,
   sensitive-permission declarations longer), AppGallery ~1–4 business days, RuStore ~1–3
   business days. These are planning estimates — verify against each console's current stated
   SLA and your own rolling history.

## 5. Listing consistency policy

- **Same brand everywhere**: identical app name root, icon, and visual system across stores —
  divergent branding fragments search equity and looks like cloning to reviewers.
- **Store-tuned keywords**: re-run keyword research per store
  (`04-aso/workflows/keyword-research.md`) — AG/RuStore have smaller indexes and stronger
  exact-match behavior (`08-knowledge/aso/ranking-factors.md` §9); RuStore listings in Russian,
  AG listings localized to its strong regions.
- **Claims parity with capability**: a flavor lacking a feature (e.g., no cloud sync in the
  GMS-less AG build) must not inherit the Play listing's claim — AG review checks this.
- **One listing source of truth** per app: maintain the master in
  `docs/factory/assets/` using `09-templates/store-listing.md`, with per-store variant sections,
  so updates propagate deliberately rather than drifting.

## 6. Quick-reference: capability substitution map

| Capability | Play build | AppGallery build | RuStore build |
|---|---|---|---|
| Billing | Play Billing Library | HMS IAP Kit | RuStore pay SDK |
| Push | FCM | HMS Push Kit | RuStore Push / FCM fallback |
| Ads | AdMob (+MAX mediation) | HMS Ads Kit / AppLovin | Yandex Ads (+AppLovin) |
| In-app review | Play In-App Review API | AG rating API | RuStore review API |
| In-app update | Play In-App Updates | AG Connect update API | RuStore update SDK |
| App distribution analytics | Play Console | AppGallery Connect | RuStore Console |

Details and current SDK coordinates: `04-aso/stores/*.md` and
`08-knowledge/monetization/ads-patterns.md` / `08-knowledge/monetization/subscription-patterns.md`.
Verify SDK versions against official docs at implementation time — they drift quarterly.

## 7. Cross-references

- Per-store ranking deltas: `08-knowledge/aso/ranking-factors.md` §9
- Per-store policy deltas: `08-knowledge/stores/policy-landmines.md` §6
- versionCode scheme: `10-releases/app-versioning-policy.md`
- Release procedure: `10-releases/release-workflow.md`
- Store operational guides: `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
  `04-aso/stores/rustore.md`
