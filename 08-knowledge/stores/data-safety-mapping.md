# Data-Safety SDK Mapping

The **reusable knowledge half** of the Data-safety problem: the question model and a per-SDK
reference for what common SDKs collect, share, and why — so an agent filling Google Play's Data
safety form can map each SDK in a target app to honest answers (D16). This file is doctrine and a
lookup table; the **procedure** — generating, reconciling, and submitting the form — is the workflow
half, `04-aso/workflows/data-safety-listing.md`. The two are designed to be read together.

**Last reviewed: 2026-06-10.** SDK data behavior, the Data-safety question set, and store
requirements all change frequently — **always verify against the SDK's own published
data-disclosure** (Google ships an official "Data safety" guidance doc per SDK; AdMob, Firebase, and
the AppLovin/Yandex SDKs publish their own collected-data tables) and against the **current Play
Console form** before submission. When this table and the SDK's live disclosure disagree, the SDK's
disclosure wins. Policy context for *why* accuracy is enforced is in `policy-landmines.md`.

---

## 1. The Data-safety question model

Play's Data safety section asks, for each **data type** the app touches, a fixed set of questions.
Answer them per data type, not per SDK — but you derive the answers by summing every SDK's behavior
(§2) plus the app's own code.

| Question | What it means | Common trap |
|---|---|---|
| **Is this data type collected?** | Does the app/SDK transmit it off the device? On-device-only processing is **not** collection. | Marking on-device work as "collected" over-discloses; transmitting then forgetting to declare under-discloses (enforced). |
| **Is it shared?** | Transferred to a **third party** (a different company). An ads/analytics SDK sending data to its own vendor is usually "collected" by you and may be "shared" depending on the vendor's role. | Ad SDKs frequently mean **shared = yes** (data goes to the ad network and its partners). |
| **Purpose(s)** | Why: app functionality, analytics, advertising/marketing, fraud prevention, personalization, account management, developer communications, etc. Multiple allowed. | Ads SDKs add "Advertising or marketing" + often "Fraud prevention"; analytics add "Analytics". |
| **Optional or required?** | Can the user use the app without this collection (e.g. consent-gated ads vs. mandatory crash reporting)? | Consent-mode / opt-out ads → "users can choose"; always-on telemetry → "required". |
| **Encrypted in transit?** | Is the data encrypted on the wire (TLS)? Reputable SDKs: yes. | You must be able to assert it truthfully — confirm the SDK uses HTTPS. |
| **Can users request deletion?** | Is there a deletion mechanism / in-app account-deletion path? | Ties to the account-deletion policy in `policy-landmines.md`; provide a real path if accounts exist. |

Data **types** are grouped by category (Location, Personal info, Financial info, App activity, App
info & performance, Device or other IDs, etc.); the most common for utility apps are **Device or
other IDs** (advertising ID), **App activity** (interactions, in-app events), **App info &
performance** (crash logs, diagnostics), and **Approximate location** (from ad SDKs).

---

## 2. Common-SDK mapping table

Reference only — **verify against each SDK's published data-disclosure** before you submit; vendors
change behavior, and your *configuration* (consent mode, ad personalization off, analytics
collection disabled) changes the answers. "Shared" here means transferred to the SDK vendor as a
third party from the app's perspective.

| SDK | Data typically collected | Shared (3rd party)? | Purposes | Notes — verify against the SDK's disclosure |
|---|---|---|---|---|
| **Google AdMob** (ads) | Advertising ID / device IDs, approximate location, app activity, diagnostics | **Yes** (ad network + partners) | Advertising/marketing, analytics, fraud prevention | Requires `AD_ID` permission declaration (API 33+) — see `policy-landmines.md`. Non-personalized/consent mode narrows purposes but rarely to zero. |
| **Firebase Analytics** (Google Analytics for Firebase) | App activity (events, screens), device/advertising IDs, approximate location, device info | Usually **no** (processed for you as a service provider) — confirm per config | Analytics (and advertising if linked to Ads/Audiences) | Linking to AdMob/Ads or enabling Google-signals shifts purposes toward advertising and can make it "shared". |
| **Firebase Crashlytics** (crash reporting) | Crash logs, diagnostics, device info, app activity; an installation UUID (Crashlytics installation ID) | Usually **no** (service provider) | App functionality / **Analytics** (app info & performance / diagnostics) | Typically "required" (always-on telemetry). Avoid attaching PII to custom keys/logs. |
| **AppLovin / MAX** (ad mediation) | Advertising ID / device IDs, approximate location, app activity, diagnostics | **Yes** (mediated networks are third parties) | Advertising/marketing, analytics, fraud prevention | Mediation means **each mediated network** also collects — disclose the union across all adapters, not just MAX. |
| **Yandex Mobile Ads** (ads) | Advertising/device IDs, approximate location, app activity, diagnostics | **Yes** | Advertising/marketing, analytics, fraud prevention | Common for the RU market; verify against Yandex's own disclosure and RuStore's equivalent (§4). |
| **Yandex AppMetrica** (analytics) | App activity, device/advertising IDs, approximate location, diagnostics/crash data | Usually **no** (service provider) — confirm | Analytics (and advertising features if enabled) | The RU-market analytics analog of Firebase; same care on linked-advertising features. |
| **Google Play Billing** (purchases) | Purchase history / financial-transaction info | No (Google processes payment; you receive purchase state) | App functionality (manage purchases/entitlements) | You typically declare purchase history under Financial info; Google handles the payment instrument — you don't collect card data. |

If an SDK isn't listed, find its **official data-safety guidance / collected-data table** and map it
through the §1 questions the same way. Never guess.

### Configuration changes the answer

The same SDK yields different honest answers depending on how the app ships it. Common levers that
*narrow* a disclosure — declare what the build actually does, not the SDK default:

- **Consent / UMP**: ad SDKs behind a working consent flow (and "non-personalized ads" when consent
  is withheld) move ad-targeting purposes toward optional and reduce ID collection in the no-consent
  path — but rarely to zero (fraud-prevention and basic delivery often remain).
- **Disable advertising ID**: setting AdMob/Analytics to not use the advertising ID (or omitting the
  `AD_ID` permission with `tools:node="remove"`) removes "Device or other IDs" for that SDK — only
  legitimate if no SDK actually reads it (verify the merged manifest, `02-engineering/06-manifest-audit.md`).
- **Analytics collection off by default until consent**: shifts App-activity collection to optional.
- **Crashlytics / telemetry**: usually *required* and always-on; honestly mark it required rather
  than inventing an opt-out that doesn't exist.

---

## 3. Worked example — an ad-funded utility app

A typical portfolio app (file recovery, AdMob via MAX mediation, Firebase Analytics + Crashlytics,
no login) sums to roughly this declaration — illustrative, **derive your own from the live inventory**:

| Data type | Collected | Shared | Purposes | Optional? | Driven by |
|---|---|---|---|---|---|
| Device or other IDs (advertising ID) | Yes | Yes | Advertising, analytics, fraud prevention | Optional (consent) | AdMob/MAX, Firebase |
| App activity (in-app events, interactions) | Yes | No* | Analytics (advertising if Ads-linked) | Optional | Firebase Analytics, ad SDKs |
| App info & performance (crash logs, diagnostics) | Yes | No | App functionality / diagnostics | **Required** | Crashlytics |
| Approximate location | Yes | Yes | Advertising | Optional (consent) | ad SDKs |

\* "No" only if Analytics isn't linked to Ads/Audiences and Google-signals is off — otherwise it
shifts toward shared/advertising. Encryption-in-transit: yes for all of the above (HTTPS). Deletion:
provide a path even without accounts (e.g. reset advertising ID guidance + a contact for data
requests). The absence of a login means **no** Personal info / Financial info rows from app code —
unless Play Billing is present, which adds a Financial-info (purchase history) row (§2).

---

## 4. The reconciliation rule (the form must match real traffic)

Play runs **automated traffic analysis** of the shipped binary and compares it to the declared form;
a mismatch is enforced (warning → update block → removal — `policy-landmines.md`). Therefore:

1. **The form is generated from the SDK inventory, not from memory.** Build it from the actual,
   current SDK list and versions in `02-engineering/04-sdk-inventory.md` (which records each SDK,
   its purpose, and the data it touches). The inventory is the source of truth for *which* SDKs are
   present; this doc maps each to *what it discloses*.
2. **Sum across every SDK and the app's own code**, per data type — including transitive SDKs and
   every mediated ad network (§2 AppLovin note).
3. **Reflect your configuration.** Consent mode, "disable advertising ID", analytics-collection
   flags, and personalization toggles change the honest answer — declare what the app *actually
   ships with*, not the SDK's default.
4. **Re-verify on every SDK add/update.** A new SDK or a major version bump can change collected
   data; the form is re-derived, not assumed stable. This is a standing P7/P8 step.
5. **Claims must match user-facing copy.** Trust lines like "Nothing is uploaded"
   (`mobile-ux-principles.md` §9) must be true *and* consistent with the form.

The executable steps — running the inventory, building the per-type answers, the pre-submission
diff against traffic — are in `04-aso/workflows/data-safety-listing.md`.

---

## 5. AppGallery & RuStore privacy-declaration equivalents

Play's Data safety form is not unique — the other stores require parallel disclosures, and the same
SDK mapping feeds them (store mechanics in `04-aso/stores/huawei-appgallery.md` and
`04-aso/stores/rustore.md`):

- **Huawei AppGallery**: requires a **privacy declaration** plus a hosted **privacy policy URL**, and
  (for apps using HMS/ad SDKs) declarations of personal-data processing. The *content* maps from the
  same §2 SDK behavior; the *form/UI* differs. Apps honoring a no-GMS claim must keep ad/analytics
  SDK choices consistent with the AppGallery build (e.g. Petal/HMS ads, AppGallery-side analytics).
- **RuStore**: requires a **privacy policy** and data-processing disclosure consistent with Russian
  data-protection requirements; RU-market SDKs (Yandex Ads / AppMetrica, §2) are the common sources,
  and moderation checks declared processing against actual behavior. RU-language policy text is
  expected.

Rule: derive **one** SDK-data map (§2/§4) and translate it into each store's declaration format —
never let the stores' declarations drift apart, because they describe the same binary's behavior.

---

## 6. Cross-references

- The workflow half (generate, reconcile, submit): `04-aso/workflows/data-safety-listing.md`
- SDK inventory (source of truth for which SDKs are present): `02-engineering/04-sdk-inventory.md`
- Why accuracy is enforced (penalties, detection): `08-knowledge/stores/policy-landmines.md`
- `AD_ID` permission declaration: `08-knowledge/stores/policy-landmines.md`, manifest audit `02-engineering/06-manifest-audit.md`
- Trust-copy consistency: `08-knowledge/design/mobile-ux-principles.md` §9
- Cross-store mechanics: `08-knowledge/stores/store-comparison.md`, `04-aso/stores/`
