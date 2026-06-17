# QR & Barcode Scanner Playbook

Category playbook for QR / barcode scanner and generator apps (QR readers, barcode scanners,
"scan & generate", product/price lookup scanners). Load during P1 per `06-playbooks/README.md`
and keep loaded through release. This is a **commodity category**: modern Android camera apps and
Google Lens scan QR codes natively, so a standalone scanner must justify its existence with
something the OS does not do — history, batch, generation, product lookup — against a user base
with **low willingness to pay** and an **ad-dominant** monetization reality.

## Category Snapshot

- **Market reality.** Enormous install volume but commoditized at the core. Since roughly Android
  9–10, the stock camera (and Google Lens) detects and opens QR codes with no app at all — which
  collapsed the "I need an app to scan a QR" reason-to-install. What survives: scanners that add
  **history** (re-find a scanned code), **batch** scanning, **code generation** (make a QR/barcode
  to share), **product/price lookup**, and **bulk inventory** use cases. The field is a sea of
  near-identical ad-funded apps plus a few polished ones.
- **User intent.** Mostly transactional and one-shot: "scan this code, do the thing" (open a URL,
  add Wi-Fi, see a menu, read a product). The retainable intents are narrower: people who
  **generate** codes (small businesses, event organizers), people who **scan in bulk** (inventory,
  events), and people who want a **history** of what they scanned. "What do I do with this result?"
  is the real UX problem, not the scanning itself.
- **Competition shape.** A few quality apps and a vast ad-stuffed tail. Reviews cluster on two
  things: "works fast / does what the camera won't" (positive) and **"ad after every single scan"**
  (the category's defining 1★ complaint). The OS-already-does-this objection means the listing has
  to *teach* the differentiator, not assume the user knows they need an app.
- **Retention reality.** Pure scanning has near-zero retention (the OS competes for free).
  Retention lives in generate + history + product lookup; without one of those, this is a
  single-use install funded entirely by the first-session ad.
- **Willingness to pay is low.** Users will not subscribe to scan a QR code. Monetization is ads
  with a small remove-ads/pro IAP — say this plainly and don't design a subscription business here.
- **Competitive reference set** (study during P1 and `04-aso/workflows/competitor-analysis.md`):

| App | What it proves | What to copy / avoid |
|---|---|---|
| Google Lens / stock camera | The free baseline that ate the core use case | Differentiate on history/batch/generate; don't compete on bare scanning |
| QR & Barcode Scanner (Gamma Play) | Clean, fast, history + generate, restrained ads = durable rank | Copy: result action sheet, history, fair ads |
| Binary Eye (open source) | A no-ads scanner can earn loyalty and 4.7★ | Copy: no-nonsense UX. Note: hard to monetize |
| Ad-stuffed clone tail | Interstitial-after-every-scan = 1★ avalanche | The exact failure to avoid |
| "QR + virus scan" clones | Fake security claims attract installs then enforcement | Never make security/antivirus claims you can't back |

## Engineering Patterns

### Scanning engine — the central decision

| Option | What it gives | License / size | Verdict |
|---|---|---|---|
| **ML Kit Barcode Scanning** (`com.google.mlkit:barcode-scanning`, unbundled `play-services-mlkit-barcode-scanning`) | Fast, robust detection of all common 1D/2D formats; auto-zoom; handles rotation/poor light well | Free; ML Kit terms; small (unbundled downloads the model from Play services) — **GMS-dependent** | **The default for a Play build.** Best detection quality for least code. Unbundled keeps APK small; bundled if offline first-run matters. |
| **ZXing** (`com.google.zxing:core` + `journeyapps:zxing-android-embedded`) | Pure-Java decode + **encode (generate codes)**; no GMS dependency | **Apache 2.0** — clean for closed source; modest size; works on non-GMS stores | **The default for generation, and the scanner of choice for AppGallery/RuStore/non-GMS builds.** Detection is good but generally below ML Kit in hard conditions. |

**Decision rule:** Play/GMS build → ML Kit for scanning, **ZXing for generating** (ML Kit doesn't
write codes). Non-GMS stores → ZXing for both. Abstract behind a `CodeScanner` interface so the
engine is flavor-selectable.

### Camera pipeline

- **CameraX** (`androidx.camera`, Apache 2.0): `Preview` + `ImageAnalysis` feeding the scanner
  analyzer; throttle analysis (`STRATEGY_KEEP_ONLY_LATEST`) and stop on first confident detection
  to avoid double-firing. Provide a torch toggle (dim codes) and tap-to-focus. Request `CAMERA`
  only. Offer "scan from image" (decode a QR in a gallery photo) — handy and only needs
  `READ_MEDIA_IMAGES` on demand.

### Format coverage

- Cover the formats users actually meet: **QR**, Data Matrix, Aztec, PDF417 (boarding passes,
  driver's licenses), and 1D retail codes **EAN-13/EAN-8, UPC-A/UPC-E, Code 128, Code 39, ITF**.
  Don't enable every exotic symbology by default — it slows detection; gate rare ones behind a
  setting. ML Kit and ZXing both cover this set.

### Generate codes

- **ZXing writer** (`MultiFormatWriter` → `BitMatrix` → bitmap) generates QR (URL, text, Wi-Fi,
  contact/vCard, email, SMS, geo, calendar event) and the common 1D formats. Offer share/save
  (SAF `ACTION_CREATE_DOCUMENT` for the PNG), size/error-correction control for QR, and a small
  template set (Wi-Fi join code, contact card) — generation is a real retention/differentiator
  feature and ZXing makes it nearly free.

### Batch & history (the retention substance)

- **Room** stores scan/generate history: raw value, parsed type (URL/Wi-Fi/product/text…),
  timestamp, format, and a user label/favorite flag. History is the #1 reason a user keeps the app
  over the stock camera — make it searchable and exportable (CSV via SAF; export is a fair pro
  feature). **Batch mode** (keep scanning, accumulate a list) serves inventory/event use cases and
  is a clean pro differentiator.

### Product lookup (a differentiator with API discipline)

- Map a scanned EAN/UPC to product info via **Open Food Facts** (open data, free — **requires
  attribution**, respect rate limits and cache aggressively in Room) for grocery/food, and/or a
  general UPC database API (most are rate-limited/keyed/paid — check terms, cache, and degrade
  gracefully when over quota or offline). Never block the scan result on a network call: show the
  raw code instantly, then enrich with product data when it arrives. Disclose any lookup call in
  the Data safety form (the scanned code/value leaves the device).

### Deep-link / URL safety (a security pattern, not optional)

- A scanned QR commonly contains a **URL — and a malicious QR is a real attack vector**
  (phishing/"quishing", drive-by, payment redirects). **Never auto-open a scanned URL.** Show the
  full, un-shortened destination, warn on suspicious patterns (IP-literal hosts, punycode/homograph
  domains, mismatched display text, non-https payment-looking links, app-deep-links), and require
  an explicit "Open" tap. Expand shortened links and show the final host before opening. This both
  protects users and is a legitimate trust/differentiator angle — but frame it as "preview before
  you open", **never** as "virus scan" (a claim you cannot back; see pitfalls).

### Architecture sketch (aligns with `08-knowledge/android/modern-stack.md`)

- `:core:scan` — `CodeScanner` interface (ML Kit + ZXing impls, flavor-selected).
- `:core:generate` — ZXing writer wrapper. `:core:history` — Room + CSV export.
- `:core:lookup` — product-lookup clients with caching + rate-limit handling.
- `:core:safety` — URL parsing/normalization + risk heuristics for the result sheet.
- `:feature:scan` / `:feature:generate` / `:feature:history` — Compose UI; Hilt DI.

## Design Patterns

- **Instant-scan hero.** App opens straight to the live camera (after a one-time camera-permission
  rationale) — scanning is the primary job; bury nothing behind a menu. Detection should feel
  instant; on detect, present the result immediately.
- **The "what do I do with the result?" action sheet** is the real design problem (the OS can scan;
  it can't always *act* well). On detection, show a bottom sheet keyed to the parsed type:
  - URL → **show the full destination + safety preview**, then Open / Copy / Share / Search.
  - Wi-Fi → "Connect to network <SSID>" one-tap join.
  - Contact/vCard → "Add to contacts".
  - Plain text → Copy / Share / Search the web.
  - Product (EAN/UPC) → product card (name/image/price if looked up) + "Search online".
  - Email/SMS/geo/calendar → the matching intent.
  Copy/Share/Search are the universal fallbacks for any result.
- **History list** — reverse-chronological, searchable, type icons, favorite/label, swipe to
  delete, tap to re-open the action sheet. The retention surface.
- **Generate flow** — a type picker (URL/text/Wi-Fi/contact/…), a simple form, live preview, then
  share/save. Keep it two screens deep, not buried.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Scan (home) | The core action | Live camera, torch, tap-to-focus, scan-from-image, batch toggle |
| Result action sheet | Turn a code into an action | Type-aware actions; URL safety preview; Copy/Share/Search fallback |
| History | Retention + re-find | Searchable, type icons, favorite, export (pro) |
| Generate | Differentiator | Type picker, form, live preview, share/save |
| Settings | Control | Formats enabled, beep/vibrate, auto-copy, ad/remove-ads, lookup on/off |

- **Accessibility floor:** content descriptions on scan controls; results readable by TalkBack;
  the URL-safety warning must be conveyed non-visually too. Run
  `03-design/09-accessibility-review.md` against scan + result-sheet.

## ASO Patterns

- **Keyword themes by intent tier** (build the full map in P7 via
  `04-aso/workflows/keyword-research.md`):

| Tier | Examples | Volume | Competition | Role |
|---|---|---|---|---|
| Head | "qr scanner", "qr code scanner", "barcode scanner", "qr code reader" | Very high | **Brutal + low differentiation** (the OS competes) | Title presence; rank is a slow velocity game |
| Function | "qr code generator", "scan barcode", "qr reader", "scan qr code" | High | High | Short + long description |
| Differentiator / long-tail | "barcode scanner with history", "batch qr scanner", "wifi qr code generator", "price scanner", "inventory barcode scanner" | Low–medium each | Weaker | **The realistic ranking entry point** and the honest reason-to-install |
| Product lookup | "product scanner", "food barcode scanner", "price check scanner" | Medium | Medium | Differentiator coverage; ties to lookup feature |

- **Handle the "the OS already does this" objection in the listing itself.** The screenshots and
  short description must answer "why install this when my camera scans QR?": lead with the
  differentiator — **history, batch, generate, product lookup, safe-link preview** — not with
  "scan QR codes". A listing that only promises scanning loses to the stock camera and converts
  poorly.
- **Title formula:** `QR & Barcode Scanner — <differentiator>` (e.g. "QR & Barcode Scanner —
  History & Generate") or `<Brand>: QR Reader & Generator`. Pack head terms into title + short
  description; carry the differentiator phrasing in the long description.
- **Screenshot messaging:** 1 = instant scan with a clean result action sheet ("Scan + act in one
  tap"); 2 = **generate a QR** ("Create QR codes to share"); 3 = history list ("Every code you've
  scanned, saved"); 4 = safe-link preview ("See where a link goes before you open it"); 5 = product
  lookup / batch. Show the differentiators, because bare scanning doesn't sell against the OS.
- **Localization:** "QR" stays Latin everywhere; "scanner"/"barcode"/"generator" translate
  predictably — run `04-aso/workflows/localization.md` for top utility locales (en, es-419, pt-BR,
  hi, id, de, fr, ru). Cheap long-tail multiplier.

- **Category benchmark (est.):** store-listing **CVR ~30–40%** — the head-term query is dead-simple
  high intent, so raw CVR is high, but install *quality* is low (single-use, ad-funded). **Rating
  norm ~4.2–4.5** and dominated by ad-experience: the apps that hold the top are the ones with
  restrained ads, while interstitial-after-every-scan clones sit at 3.x. Read CVR alongside
  retention/ARPU, not in isolation — high CVR here does not mean a healthy business. Record
  actuals against this in the ASO_HANDOVER Baseline Metrics Snapshot (`_V11_SPEC` D16).

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1):**

```text
qr scanner, qr code scanner, qr code reader, barcode scanner, qr reader,
scan qr code, scan barcode, qr code generator, barcode reader, qr code maker,
wifi qr code, qr generator, barcode generator, scan to connect wifi,
barcode scanner with history, batch qr scanner, product scanner, price scanner,
food barcode scanner, upc scanner, ean scanner, data matrix scanner,
pdf417 scanner, scan qr from image, create qr code, qr code creator, scan and generate
```

## Monetization Patterns

- **This category is ad-dominant.** Willingness to pay is low; revenue is overwhelmingly ads with
  a small **remove-ads / pro IAP**. Design the ad placement as the core monetization and the IAP as
  the relief valve — not the other way round.
- **The cardinal rule: NOT an interstitial after every scan.** Interstitial-after-every-scan is the
  category's defining review-bomb cause ("ad every time I scan, uninstalled"). Cap hard:
  **interstitial at most 1 per N scans with a generous minimum spacing** (e.g. 1 per 4–5 scans,
  ≥ 90 s apart) and **never** between the scan and the result (the user has a task in hand). A
  banner on the scan/result/history screens is the low-friction always-on surface.
- **Remove-ads IAP** is the primary purchase: a cheap one-time unlock that removes all ads. **Pro
  = no ads + batch + history export (CSV) + maybe bulk/inventory tools.** Keep core scanning,
  result actions, and basic generate **free** — gating the core behind a paywall in a
  commoditized, OS-substitutable category just sends users back to the stock camera.
- **Low subscription viability — say so.** Do not build a subscription business here; users won't
  recurring-pay to scan QR codes, and a sub paywall in this category breeds resentment and refunds.
  A one-time remove-ads (and optional one-time "pro tools") unlock is the right model. If the
  operator insists on a subscription, flag the mismatch explicitly in P1 and expect low conversion.
- **Placement matrix:**

| Placement | Verdict | Rationale |
|---|---|---|
| Banner on scan / result / history screens | Allowed | Always-on, low-friction surface |
| Interstitial between scan and result | **Forbidden** | Task-blocking; pure churn |
| Interstitial after a scan, hard-capped (≤ 1 per 4–5 scans, ≥ 90 s) | Allowed, capped | The only safe interstitial slot; over this = 1★ avalanche |
| Interstitial after every scan | **Forbidden** | The category's #1 review-bomb cause |
| Any ad over the live camera preview | **Forbidden** | Obscures scanning; mis-taps; accidental-click policy |
| App-open ad | Caution, cap 1/session | App is opened to scan *now*; suppress when nearing the limit |
| Rewarded ad for a one-off pro action (e.g. one CSV export) | Allowed | Honest exchange; paywall A/B lever |

- Ad-SDK note: mediation bloat is a real risk here (see pitfalls) — every added network drags in
  permissions and SDK weight. This section overrides
  `08-knowledge/monetization/ads-patterns.md` where they conflict for this category.

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| Auto-opening a scanned URL (malicious-QR / "quishing" liability) | Security / users harmed | Phishing/redirect harm; trust + policy exposure | Never auto-open; show full destination, expand shorteners, warn on suspicious hosts, require explicit Open |
| Interstitial after every scan (or between scan and result) | Review-bomb | 1★ avalanche; uninstalls back to the stock camera | Hard cap (≤ 1 per 4–5 scans, ≥ 90 s); never pre-result; verify in P8 |
| Ad-SDK permission bloat (mediation pulling location/phone-state/etc.) | Privacy / policy / trust | Over-permissioned scanner looks like spyware; Data-safety mismatch | Minimize networks; audit transitive permissions in P3; declare every data flow |
| Fake "virus scan" / "antivirus" / "security scanner" claims | Play policy (misleading functionality) | Misleading-claims suspension | Frame URL safety as "preview before you open"; never claim malware/virus scanning |
| Competing on bare scanning vs the OS | Strategy/ASO | No reason-to-install; poor retention; weak CVR-to-value | Lead with history/batch/generate/lookup/safety in product and listing |
| GMS-only ML Kit shipped to AppGallery/RuStore | Technical/distribution | Scanner absent on non-GMS devices | ZXing flavor for non-GMS stores (also handles generation everywhere) |
| Product-lookup API misuse (no caching, no attribution, ignoring rate limits) | Ops / legal | API bans, broken lookups, attribution-license breach | Cache in Room, respect limits, attribute Open Food Facts, degrade gracefully offline |
| Over-permissioning (location/contacts) for "features" | Privacy/policy | Trust collapse; Data-safety findings | Request `CAMERA` only (+ `READ_MEDIA_IMAGES` on demand for scan-from-image) |
| Enabling every exotic symbology by default | Performance | Slow/janky detection | Default to common formats; gate rare ones in settings |
| Cloned-template signals (boilerplate clone UI + listing) | Play policy (spam/min functionality) | De-indexing / rejection as repetitive | Differentiated UI in P6, original screenshots/copy in P7 |
| Treating it as a subscription business | Monetization | Low conversion, refund/resentment reviews | Remove-ads / one-time pro model; flag any sub insistence in P1 |

## Recommended Workflow

1. **P1 Discovery** — classify the existing app's scan engine (ML Kit vs ZXing vs other), GMS
   dependency vs target stores, whether it has the differentiators (history/batch/generate/lookup),
   its URL-handling behavior (does it auto-open? — a security finding), permission set (any
   location/contacts/phone over-ask is a flag), and its ad cadence. Surface the OS-commoditization
   reality and the differentiator strategy to the operator up front.
2. **P3 Engineering Audit** — ad-SDK / transitive-permission audit (the bloat risk); URL-safety
   review (auto-open = Critical security finding); license + GMS-dependency review; product-lookup
   API discipline (caching/attribution/limits); battery/perf of the analysis loop.
3. **P4 Modernization** — ML Kit (Play) / ZXing (generate + non-GMS) behind a `CodeScanner`
   interface; type-aware result action sheet with URL-safety preview; Room history + CSV export;
   optional product lookup with caching; ZXing generation.
4. **P5/P6 Design & Redesign** — instant-scan hero, the result action sheet, history, and generate
   are the whole product; the action sheet quality is the rating driver.
5. **P7 ASO** — differentiator-led keyword map and screenshots that answer the
   "OS-already-does-this" objection; never make security/antivirus claims.
6. **P8 Verification & Release** — URL-safety behavior tests (no auto-open; suspicious-link
   warnings fire), ad-cadence verification (no per-scan interstitial; spacing enforced),
   permission/Data-safety consistency, multi-format scan corpus, generation round-trip.

Category-specific P8 verification additions (append to
`07-checklists/release-readiness.md` items):

```text
[ ] No scanned URL ever auto-opens; full destination shown; shorteners expanded before Open
[ ] Suspicious-link warning fires on IP-literal / punycode / mismatched-host test QRs
[ ] No interstitial between scan and result; interstitial cap (≤1 per 4–5 scans, ≥90s) enforced
[ ] Only CAMERA (+ on-demand READ_MEDIA_IMAGES) requested; no transitive location/phone perms
[ ] Multi-format corpus (QR/EAN/UPC/Code128/PDF417/DataMatrix) all decode
[ ] Generate→scan round-trip: a generated QR/barcode scans back to its source value
[ ] No "virus/antivirus/security scan" claims in metadata or UI
[ ] Product lookup caches results and degrades gracefully offline / over rate limit
```

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`.
