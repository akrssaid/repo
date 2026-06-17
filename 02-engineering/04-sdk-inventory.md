# Third-Party SDK Inventory Method

This method finds and documents every third-party SDK embedded in the target app — ads, analytics,
crash reporting, attribution, billing, push — because each one carries version risk, privacy/
Data-safety obligations, init-time cost, and store-policy exposure. It runs in P1 (seed list from
the Application class) and fully in P3 under `01-core/prompts/03-modernization-audit.md`; its
Data-safety table is re-consumed at P8 by `01-core/prompts/16-release-preparation.md`. An SDK that
appears in the dependency graph but never initializes still ships its manifest entries and data
collection — inventory it anyway and flag it for removal.

## Detection: Three Signals, All Required

For each SDK gather (1) **dependency** evidence from the graph dumps
(`docs/factory/audits/deps-<config>.txt`, produced by `02-engineering/03-dependency-analysis.md`),
(2) **manifest** evidence from the merged manifest (see `02-engineering/06-manifest-audit.md` for
how to obtain it), and (3) **initialization** evidence via the grep patterns below. A dependency
with no init code = dead weight (removal candidate). Init code with no current dependency =
leftover dead code.

### Grep Patterns per SDK

Run from repo root; add `--include="*.java"` where the codebase has Java.

```bash
# AdMob / Google Mobile Ads
grep -rn "MobileAds.initialize\|com.google.android.gms.ads" --include="*.kt" --include="*.xml" app/src
grep -n "com.google.android.gms.ads.APPLICATION_ID" app/src/main/AndroidManifest.xml

# Firebase (umbrella — note which products)
grep -rn "com.google.gms.google-services" app/build.gradle* build.gradle* settings.gradle*
ls app/google-services.json 2>/dev/null && echo "google-services.json present"
grep -rn "FirebaseApp.initializeApp\|Firebase\." --include="*.kt" app/src | head -20

# Crashlytics
grep -rn "FirebaseCrashlytics\|firebase.crashlytics\|fabric" --include="*.kt" --include="*.gradle" --include="*.kts" app/

# AppLovin (MAX)
grep -rn "AppLovinSdk\|applovin" --include="*.kt" --include="*.xml" app/src
grep -n "applovin.sdk.key" app/src/main/AndroidManifest.xml

# Yandex Ads
grep -rn "com.yandex.mobile.ads\|MobileAds.initialize" --include="*.kt" app/src   # Yandex also names it MobileAds — check imports

# AppMetrica (analytics + attribution)
grep -rn "AppMetrica.activate\|YandexMetrica.activate\|io.appmetrica" --include="*.kt" app/src

# Google Analytics / Firebase Analytics
grep -rn "FirebaseAnalytics.getInstance\|logEvent(" --include="*.kt" app/src | head -10

# Play Billing
grep -rn "BillingClient.newBuilder\|com.android.billingclient" --include="*.kt" app/src
grep -n "com.android.vending.BILLING" app/src/main/AndroidManifest.xml

# RuStore Billing / RuStore SDKs
grep -rn "RuStoreBillingClient\|ru.rustore.sdk" --include="*.kt" --include="*.kts" app/

# Attribution (AppsFlyer / Adjust / Branch)
grep -rn "AppsFlyerLib.getInstance\|Adjust.onCreate\|Branch.getAutoInstance" --include="*.kt" app/src

# Push (FCM / HMS)
grep -rn "FirebaseMessagingService\|HmsMessageService" --include="*.kt" --include="*.xml" app/src

# Generic catch-all: everything initialized in the Application class
grep -rn --include="App*.kt" "init\|initialize\|activate\|start" app/src/main
```

PowerShell translation for any pattern above (one shape covers them all):

```powershell
# grep -rn "PATTERN" --include="*.kt" --include="*.xml" PATH  →
Get-ChildItem -Recurse app\src -Include *.kt,*.xml | Select-String "PATTERN"
# Single-file greps (manifest checks)  →
Select-String -Path app\src\main\AndroidManifest.xml -Pattern "PATTERN"
# File-presence checks (e.g. google-services.json)  →
Test-Path app\google-services.json
```

Also check `ContentProvider`-based auto-init: many SDKs (Firebase, WorkManager, AppMetrica) start
via merged-manifest providers — list `<provider>` entries in the merged manifest and attribute each
to an SDK. `tools:node="remove"` on an `androidx.startup` or SDK provider means someone already
disabled auto-init; find where manual init happens instead.

## Per-SDK Record (fill for every SDK found)

| Field | What to record |
|---|---|
| SDK + vendor | e.g., AppLovin MAX |
| Purpose | Ads mediation / analytics / crash / attribution / billing / push |
| Artifact + version | exact coordinates from graph dump; latest stable alongside |
| Init location | file:line (Application.onCreate, ContentProvider auto-init, lazy at first use) |
| Consent gating | Is init gated behind UMP/consent flow? (required for ads SDKs in EEA — policy exposure if not) |
| Data collected | per vendor Data-safety documentation: device IDs, ad ID, IP, usage events, crash traces |
| Manifest footprint | permissions, components, metadata it merges in |
| Kill criteria | conditions under which we remove it (see below) |

**Kill criteria — recommend removal when any holds:**

| Criterion | Example |
|---|---|
| Redundant with another inventoried SDK | Two analytics SDKs (see below) |
| No init path reachable / feature dead | Billing SDK but app has no purchases |
| Vendor sunset or store-policy conflict | Old ad SDK without UMP support; SDK on Google Play SDK Index with policy warning |
| Data collection unjustifiable for app function | Attribution SDK in an app with no paid UA |
| Version abandoned + native libs without 16 KB page-size support | Old NDK-based SDKs (blocks targetSdk 36 era compliance) |

## Redundancy Detection

Group the inventory by Purpose. More than one SDK per row below = High finding (extra app size,
duplicate data collection to declare, doubled init cost):

| Purpose | Common duplicate pairs seen in acquired apps |
|---|---|
| Analytics | Firebase Analytics + AppMetrica; Firebase + Amplitude/Mixpanel |
| Crash | Crashlytics + AppMetrica crashes + Sentry |
| Ads | Direct AdMob + AppLovin MAX both initialized (mediation should own AdMob) |
| Attribution | AppsFlyer + Adjust (one must go) |
| Push | FCM + a vendor push SDK doing its own FCM wrapping |

Decision rule: keep the SDK that (1) the monetization/analytics strategy in
`08-knowledge/monetization/ads-patterns.md` prefers, (2) has the smaller Data-safety footprint,
(3) is required by a store flavor (e.g., RuStore billing stays in the rustore flavor only —
verify it is flavor-scoped: `rustoreImplementation(...)` not `implementation(...)`).

## Size & Startup Cost Check

For every "keep" SDK, record its cost so redundancy decisions are economically grounded.
**Size**: compare AAB size with the SDK's dependency commented out (Tier 2 build), or inspect
Android Studio Build > Analyze APK… → per-library dex + native-lib contribution.
**Startup**: SDKs initialized in `Application.onCreate` add cold-start time — the measurement
method (Displayed-time capture, init-cost attribution, before/after toggling) is owned by
`02-engineering/10-performance-audit.md`; run its cold-start procedure per kept SDK rather than
improvising one here.

An SDK costing >2 MB AAB or >150 ms cold start needs its purpose row to justify it explicitly;
otherwise propose lazy init (first-use) or removal as an ENG- finding (Medium). Check each kept
SDK against the **Google Play SDK Index** (play.google.com/sdks) for policy warnings and version
advisories — an SDK version flagged there is an automatic High finding.

## Data-Safety-Form Impact Table

Every SDK maps to Google Play Data safety form declarations. Build this table; it is the
**direct input** to `08-knowledge/stores/data-safety-mapping.md` (SDK → form-section mapping) and
to the form-update workflow `04-aso/workflows/data-safety-listing.md`, and the authoritative input
for the store listing at P7/P8 (`04-aso/stores/google-play.md`, `10-releases/release-workflow.md`).
Verify against each vendor's published data-safety guidance and the Google Play SDK Index — the
table below lists the typical baseline:

| SDK | Data types typically declared | Shared with 3rd parties | Ad ID used | Notes |
|---|---|---|---|---|
| AdMob / GMA | Ad ID, coarse location (if permitted), app interactions, diagnostics | Yes | Yes | Requires `AD_ID` permission on API 33+ (merged automatically) |
| AppLovin MAX | Ad ID, IP-derived location, interactions | Yes | Yes | Mediation partners each add their own collection — declare the union |
| Yandex Ads | Ad ID, device info, interactions | Yes | Yes | Check current vendor data-safety doc; regional policy differences |
| Firebase Analytics | App interactions, Ad ID (unless disabled), diagnostics | Configurable | Default yes | `google_analytics_adid_collection_enabled=false` disables Ad ID |
| AppMetrica | Device IDs, IP, interactions, crashes | Yes | Yes | Counts as analytics + crash — affects two form sections |
| Crashlytics | Crash traces, device state, installation IDs | No (Google processor) | No | Diagnostics section |
| Play Billing | Purchase history | No | No | Financial info section |
| RuStore Billing | Purchase data (RuStore account) | RuStore | No | Only relevant to RuStore listing, not Play form — but the code ships in all flavors unless scoped |
| AppsFlyer/Adjust | Ad ID, install referrer, IP, events | Yes | Yes | Attribution = "Advertising or marketing" purpose |

A mismatch between this table and the live Data safety form is a **Critical** finding (policy
strike risk). Removing an SDK at P4 means the form must be updated at P8 — record the linkage in
the finding.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| SDK inventory table (per-SDK records) | `docs/factory/audits/ENGINEERING_AUDIT.md` "Third-Party SDKs" section | ENG- findings (stale/abandoned/redundant SDKs); P4 roadmap removal items |
| Redundancy verdicts | ENGINEERING_AUDIT.md | Monetization decisions (`08-knowledge/monetization/ads-patterns.md`); app-size reduction items |
| Data-safety impact table | ENGINEERING_AUDIT.md + copied into `docs/factory/reports/` at P8 | `08-knowledge/stores/data-safety-mapping.md`; `04-aso/workflows/data-safety-listing.md`; `01-core/prompts/16-release-preparation.md`; `08-knowledge/stores/policy-landmines.md` checks |
| Consent-gating gaps | ENG- findings (High/Critical) | P4 fixes; `07-checklists/release-readiness.md` |
| Flavor-scoping check (store-specific SDKs) | ENGINEERING_AUDIT.md | Release flavor matrix in `10-releases/release-workflow.md` |

Inventory is complete when every ads/analytics/crash/attribution/billing/push artifact in the
release graph dump has a full per-SDK record, every record has all three detection signals
resolved (dependency, manifest, init), and the Data-safety table covers every SDK marked "keep".
