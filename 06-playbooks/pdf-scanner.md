# PDF Scanner Playbook

Category playbook for document/PDF scanner apps — the "point your camera at a paper, get a
cropped, dewarped, OCR'd PDF" tools (receipt scanners, ID/document scanners, "scan to PDF",
"scan to text"). Load during P1 per `06-playbooks/README.md` and keep loaded through release.
This is a huge, crowded, freemium-heavy utility category where the engineering is largely a
solved problem (Google now ships a document-scanner API), so the win is **trust, output quality,
and a paywall users don't resent** — not novel computer vision.

## Category Snapshot

- **Market reality.** One of the largest utility categories on Google Play and a perennial
  top-grosser tier — CamScanner, Microsoft Lens, Adobe Scan, TapScanner, Genius Scan, Tiny
  Scanner, and an enormous clone tail. Demand is durable (every knowledge worker, student, and
  small business scans documents) and freemium monetization is mature, so this category has real
  revenue — and real competition. Organic installs are still winnable on intent long-tail
  ("scan document to pdf", "scan to text", "id card scanner") and on quality/trust signals.
- **User intent.** Transactional and recurring: "turn this paper into a file I can send/store."
  Sub-intents that drive feature and keyword choices: receipts (expense/tax), IDs and forms
  (sharp edges, no glare), multi-page contracts (batch + reorder), whiteboards/notes (legibility
  over fidelity), and **scan-to-text** (OCR to extract copyable/searchable content). Output is
  almost always a PDF to share; secondary output is JPG or extracted text.
- **Competition shape.** A few brand leaders with full suites (OCR in many languages, cloud sync,
  e-sign, folders), then a long freemium tail. The tail trains users to expect a **watermark on
  free exports** and a paywall — which is both the norm and the opening: an app with honest
  limits, no watermark surprise after export, and clean OCR out-reviews the resentful tail.
- **Retention reality.** Most casual users scan rarely (single receipt, one form). Retention
  comes from becoming the habitual tool for a recurring job (receipts, invoices) via a useful
  **document gallery/home** and folder organization, plus sync so scans survive a phone change.
- **Existential trust note.** CamScanner was pulled from Play in 2019 after a malicious ad-SDK
  module was found bundled in it — the category's users are measurably wary of scanner apps and
  permissions. Trust is a first-class feature here, addressed in every section below.
- **Competitive reference set** (study during P1 and `04-aso/workflows/competitor-analysis.md`):

| App | What it proves | What to copy / avoid |
|---|---|---|
| Adobe Scan | Brand + strong OCR + free tier wins trust and head terms | Copy: clean capture→OCR flow. Avoid: account-required friction |
| Microsoft Lens | Free, no-watermark, no-nag = a 4.7★ trust moat | Copy: honest free tier as positioning. The bar for "no resentment" |
| CamScanner | Feature breadth and revenue ceiling — and the malware cautionary tale | Copy: feature set. Avoid: ad-SDK greed, the trust collapse it caused |
| Genius Scan | Clean utility, fair freemium, strong edge detection | Copy: capture quality, restrained monetization |
| TinyScanner / clone tail | Watermark-on-free + interstitials = resentment reviews | The bar to clear, not the model to follow |

## Engineering Patterns

### The capture-to-PDF pipeline (five stages, each a swappable module)

`Camera → edge detect + perspective correct → image enhance → (optional) OCR → PDF assemble`.
Keep each stage behind an interface so the heavy CV stage can be swapped (see the build-vs-API
decision below) without touching capture or export.

### Edge detection & perspective correction — the central technical decision

| Option | What it gives | License / size | Verdict |
|---|---|---|---|
| **Play services Document Scanner API** (`com.google.android.gms:play-services-mlkit-document-scanner`, `GmsDocumentScanner`) | A full system-provided capture flow: live edge detection, auto-capture, crop/dewarp, filters, multi-page, and direct **PDF/JPEG output** via an `Intent`-style activity | Free; ML Kit terms; **near-zero app-size cost** (downloaded on demand from Play services) — but **requires GMS**, so it is absent on AppGallery/RuStore/de-Googled devices | **The pragmatic default for a Play build.** It hands you a polished scanner for a few dozen lines, sidesteps the hardest CV work, and Google maintains it. Trade-off: less UI control, GMS-only. |
| **ML Kit Document Scanner (bundled vs unbundled)** | Same API; choose bundled model to reduce first-use download | Larger APK if bundled | Use unbundled (download-on-demand) unless offline first-run matters. |
| **OpenCV** (`org.opencv:opencv` / native) | Full control: your own contour detection, `getPerspectiveTransform`/`warpPerspective`, custom auto-capture and filters | **Apache 2.0** (OpenCV 4.5+; older 3.x was BSD) — clean for closed source; but a **large native `.so`** (tens of MB) and you own all the CV tuning | The fallback when you need full UI control, non-GMS stores, or behavior the API won't give. Heavier to build and maintain; verify 16 KB page-size compliance of the native libs. |

**Decision rule:** Play build with GMS → start with the Play-services Document Scanner API; only
drop to OpenCV if you need pixel-level control or must ship to non-GMS stores. For a multi-store
release, abstract behind a `DocumentScanner` interface with a GMS implementation and an OpenCV
implementation selected at the flavor level.

### Camera (when you build your own capture, i.e. the OpenCV path)

- **CameraX** (`androidx.camera`, Apache 2.0) — `Preview` + `ImageAnalysis` for live edge overlay,
  `ImageCapture` for the full-res frame. `ImageAnalysis` runs your contour detector on downscaled
  frames at preview rate; capture at full resolution only on shutter/auto-capture. Lock focus and
  meter on the detected document quad. Request `CAMERA` only — never storage for capture.
- **Auto-capture UX:** detect a stable, well-framed quad for ~N frames (no motion, all four
  corners in-bounds, adequate fill), then fire automatically with a subtle countdown ring. This
  is the single biggest perceived-quality differentiator versus manual-shutter clones.

### Image processing (the "looks like a real scan" stage)

- **Filters users expect:** Original (color photo), Color document (white-balanced, contrast
  lifted), Grayscale, and **B/W (binarized)** — the default for text documents. Binarization via
  adaptive thresholding (OpenCV `adaptiveThreshold`, or the API's built-in filters) beats global
  threshold under uneven lighting.
- **Shadow/glare removal:** the #1 quality complaint. Background normalization (estimate the
  illumination field and divide it out) before thresholding removes the gradient that makes
  binarized scans look blotchy. The Play-services scanner does a competent version of this for
  free — another reason to start there.
- Process on a background thread (`Dispatchers.Default`), at capture resolution, with bitmaps
  recycled aggressively; large multi-page batches at full res are a memory hazard (see pitfalls).

### OCR — the differentiator, and the cost/privacy decision

| Path | Quality | Languages | Privacy / cost | Verdict |
|---|---|---|---|---|
| **ML Kit Text Recognition v2** (on-device, `com.google.mlkit:text-recognition`) | Strong for printed Latin; separate models for Chinese, Devanagari, Japanese, Korean | Latin bundled small; others as on-demand model downloads | **On-device, free, offline, no documents leave the phone** — the privacy story users want | **Default.** Pick the script models you advertise; download on demand. |
| **Cloud OCR** (Google Cloud Vision, Azure, etc.) | Highest accuracy, handwriting, exotic scripts, layout | Broadest | **Documents leave the device → Data safety disclosure + privacy-policy burden + per-call cost** | Only if on-device is genuinely insufficient; declare it loudly, make it optional. |
| **Tesseract** (`tess-two`/`tesseract4android`) | Good with tuned data | Many via traineddata | **Apache 2.0** but heavier and generally below ML Kit on-device quality now | Rarely worth it over ML Kit for a Play build; consider only for non-GMS stores. |

On-device ML Kit is the right default: it makes the **"your documents never leave your phone"**
trust claim *true*, which is the claim this wary category responds to. Build a **searchable PDF**
(invisible text layer over the page image) so "scan to text" and in-PDF search both work.

### PDF assembly & output

- **`android.graphics.pdf.PdfDocument`** (platform, no dependency) assembles multi-page PDFs from
  the processed page bitmaps; draw the page image, and for searchable PDFs draw the OCR text as an
  invisible/low-opacity text layer positioned from the recognition bounding boxes. For richer PDF
  features (encryption, compression tuning) a library like PDFBox-Android (Apache 2.0) is an
  option, but `PdfDocument` covers the core. Compress page JPEGs sensibly — uncompressed
  multi-page scans balloon file size and memory.
- **Output via SAF.** Export with `ACTION_CREATE_DOCUMENT` (PDF) so the user chooses the
  destination; share via `ACTION_SEND` with a `FileProvider` `content://` URI. **Do not request
  `MANAGE_EXTERNAL_STORAGE`** — a scanner's core function (camera → app-private processing →
  user-chosen export) needs no broad storage access, and the all-files declaration is routinely
  rejected for scanners (same carve-out reasoning as document readers; see
  `06-playbooks/document-reader.md`). `READ_MEDIA_IMAGES` only if you offer "scan from an existing
  photo".
- **Cloud sync (optional retention feature):** Drive/Dropbox/OneDrive via their SDKs, or an
  operator backend. Any cloud path = documents leave the device = Data safety form entry + privacy
  policy + a clear in-app disclosure; keep it opt-in. Encrypt at rest if you host.

### Architecture sketch (aligns with `08-knowledge/android/modern-stack.md`)

- `:core:scanner` — `DocumentScanner` interface (GMS + OpenCV implementations, flavor-selected).
- `:core:imaging` — filters, shadow removal, binarization (pure functions over bitmaps).
- `:core:ocr` — `TextRecognizer` abstraction over ML Kit (cloud impl behind a feature flag).
- `:core:export` — `PdfBuilder`, SAF export, share; `:core:library` — Room of documents/pages.
- `:feature:capture` / `:feature:review` / `:feature:home` — Compose UI; Hilt DI.

## Design Patterns

- **The flow is the product: capture → batch → review/reorder → export.**
  1. **Capture** — full-screen camera with live edge overlay; auto-capture with manual override;
     a page-count badge so the user knows they're building a multi-page doc.
  2. **Batch** — after each shot, return to camera to add the next page (this loop, done well, is
     why multi-page contracts get scanned in your app and not the clone's). "Add page" and "Done"
     are the two primary actions.
  3. **Review/reorder** — a page strip: per-page recrop, rotate, re-filter, delete, and
     drag-to-reorder. Reorder is a top "missing feature" complaint in clones.
  4. **Export** — name the document, choose PDF/JPG/text, pick quality, choose B/W vs color,
     then SAF save / share. OCR runs here (or lazily) for searchable PDF and scan-to-text.
- **Auto-capture UX** (covered in Engineering) is the headline design moment — a confident,
  hands-free capture feels premium; a manual-only shutter feels like a clone.
- **Filters** presented as a thumbnail row under the page preview (Auto/B&W/Grayscale/Color),
  applied live, remembered as the per-document default. B/W is the sensible default for text.
- **The gallery-of-docs home** = the retention surface: documents as cards/list with first-page
  thumbnails, page counts, dates, and folders/tags. A prominent camera FAB starts a new scan.
  Search across OCR'd text turns the home into "find that receipt", the recurring re-entry value.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Home / document gallery | Retention + re-find | Thumbnails, folders, OCR search, camera FAB |
| Capture | The core action | Live edge overlay, auto-capture, page-count badge |
| Review / reorder | Quality control | Page strip, recrop/rotate/filter/delete, drag reorder |
| Export | Conversion + output | Format, quality, B/W toggle, name, SAF save / share |
| Document detail | Reuse | View pages, re-export, OCR text, share, move to folder |
| Settings | Control + trust | Default filter, OCR languages, sync, "documents stay on device" statement |

- **Accessibility & trust floor:** content descriptions on capture controls; a plain "why we need
  the camera" rationale before the permission prompt; never request contacts/location/phone (a
  scanner asking for them is the CamScanner-trauma trigger). Run
  `03-design/09-accessibility-review.md` against the capture and review screens.

## ASO Patterns

- **Keyword themes by intent tier** (build the full map in P7 via
  `04-aso/workflows/keyword-research.md`):

| Tier | Examples | Volume | Competition | Role |
|---|---|---|---|---|
| Head | "pdf scanner", "scanner app", "document scanner" | Very high | Brutal (CamScanner, Adobe, Lens) | Title presence; long-game velocity |
| Intent / task | "scan document to pdf", "scan to pdf", "scan to text", "camera scanner" | High | High | Short + long description natural phrasing |
| Differentiator | "ocr scanner", "scan text from image", "pdf scanner with ocr" | Medium | Medium | OCR is the keyword differentiator — lead with it if your OCR is genuinely good |
| Use-case long-tail | "receipt scanner", "id card scanner", "scan id to pdf", "whiteboard scanner", "book scanner" | Low each, additive | Weaker | **The realistic ranking entry point** for a new/modernized app |

- **Intent phrasings convert better than the bare head term:** "scan document to pdf" and "scan
  to text" are buyer-intent queries — weight them in the short description and the first long-
  description sentences.
- **OCR as a differentiator keyword:** if on-device OCR is solid, make "OCR" / "scan to text" /
  "extract text from image" a front-and-center title-or-short-description term — many clones can't
  credibly claim it, so it's a wedge. Don't claim it if accuracy can't back it (see pitfalls).
- **Screenshot messaging — trust + output quality:** 1 = the auto-capture moment with a crisp
  cropped result and a trust caption ("Sharp, dewarped scans — on your device"); 2 = before/after
  of shadow removal / B/W enhancement (output quality sells here); 3 = OCR / scan-to-text with
  selectable text; 4 = multi-page batch + reorder; 5 = the document gallery / folders; 6 = a "no
  watermark / your files stay private" trust frame **only if true**. In this wary category,
  privacy and output-quality captions out-convert feature lists.
- Mirror claims with the product: "no watermark" or "documents stay on your device" in the
  listing become enforced product requirements (verify against the Data safety form in P8).
- **Localization:** the long tail translates predictably ("escáner de documentos", "scanner de
  documents", "Dokumentenscanner") — run `04-aso/workflows/localization.md` for the top reader/
  utility locales (en, es-419, pt-BR, hi, id, de, fr, ru); "PDF"/"OCR" stay Latin acronyms.

- **Category benchmark (est.):** store-listing **CVR ~25–35%** — intent is strong and commercial
  ("I need to scan this now"), so credible listings convert well; this is a higher-CVR category
  than passive utilities. **Rating norm ~4.4–4.6** — the no-watermark trust leaders (Lens, Adobe)
  sit high and set the expectation, while watermark/interstitial clones sink toward 4.0; treat
  sub-4.3 as a monetization-resentment problem to fix before ASO spend. Record actuals against
  this in the ASO_HANDOVER Baseline Metrics Snapshot (`_V11_SPEC` D16).

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1):**

```text
pdf scanner, document scanner, scanner app, scan document to pdf, scan to pdf,
scan to text, camera scanner, mobile scanner, ocr scanner, scan text from image,
extract text from image, pdf scanner with ocr, receipt scanner, id card scanner,
scan id to pdf, book scanner, whiteboard scanner, photo to pdf, image to pdf,
scan and sign, multi page scanner, scan documents, paper scanner, scan to jpg,
free document scanner, scanner pdf maker, scan business card, fax scanner
```

## Monetization Patterns

- **Freemium is the category model;** the question is *which limits don't breed resentment*.
- **Free tier (acceptable limits):** unlimited capture, all filters, basic PDF export, **on-device
  OCR limited or per-day capped**, folders. The free tier must produce a usable scan or the trust
  story collapses.
- **Pro tier (what converts):** **no watermark**, **full/batch OCR + searchable PDF**, **batch
  export / multi-page without limit**, cloud sync, e-sign, advanced filters, PDF tools
  (merge/compress/password). OCR + batch + no-watermark are the three proven pro levers.
- **The watermark-on-free pattern — and its review risk.** A footer watermark on free exports is
  the category norm and a legitimate upgrade nudge, BUT it is a top 1★ driver when it's a
  *surprise* (user scans, exports, sends to their boss, then discovers the watermark). Rules if
  you use it: disclose it **before** export (a labeled toggle/banner on the export screen, "Free
  exports include a watermark — remove with Pro"), keep it a discreet footer (never stamped across
  content), and never watermark a scan the user already paid to remove. A surprise watermark is
  the single fastest way to a resentment review wave in this category.
- **Sub vs lifetime:** subscriptions dominate the leaders (ongoing value = OCR models, cloud
  sync) — weekly/monthly/annual tiers, anchor on annual. **Offer a lifetime/one-time option too:**
  a meaningful share of scanner users distrust subscriptions for a "convert paper to PDF" utility
  and convert better on a one-time unlock; A/B it. As with any weekly tier, trial→paid terms must
  be unmissable on the paywall (Play subscription transparency) or chargebacks/"scam" reviews
  follow. Subscription detail: `08-knowledge/monetization/subscription-patterns.md`.
- **Placement matrix:**

| Placement | Verdict | Rationale |
|---|---|---|
| Banner/native on the document gallery home | Allowed | Passive browsing surface |
| Interstitial after export completes | Allowed, capped (1 per N exports, spaced) | Value delivered; never before/at capture |
| Interstitial before/at capture or before export | **Forbidden** | Task-blocking; the category's churn driver |
| Any ad over the camera preview or page content | **Forbidden** | Mis-taps ruin a capture; accidental-click policy risk |
| Rewarded ad for one watermark-free export | Allowed | Honest exchange; doubles as a paywall A/B lever |
| App-open ad | Caution, cap 1/session | Scanner is opened mid-task; suppress when launched to capture |

- This section overrides `08-knowledge/monetization/ads-patterns.md` where they conflict for this
  category.

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| Trust collapse from over-asking permissions / shady SDKs (the CamScanner-style pattern) | Reputation / policy / users wary | Uninstalls, "is this safe?" reviews, possible malware flag | Request `CAMERA` only; on-device OCR; vet every ad/analytics SDK in P3; "stays on device" story made true |
| Surprise watermark on free exports | Reviews | Resentment 1★ wave ("ruined my document") | Disclose before export; discreet footer only; never stamp content |
| OCR accuracy overclaiming | Play policy (misleading claims) / reviews | Misleading-functionality risk + refunds | Claim only languages/quality you ship; demo realistic OCR in screenshots; qualify ("printed text") |
| Requesting `MANAGE_EXTERNAL_STORAGE` | Play policy | All-files declaration rejected for a scanner; update blocked | SAF export (`ACTION_CREATE_DOCUMENT`) + FileProvider share; no broad storage |
| Cloud OCR/sync without Data safety disclosure | Play policy / privacy | Data-safety mismatch; trust damage | Declare any off-device path; keep it opt-in; privacy policy updated |
| Large-PDF / multi-page memory blowup (OOM on big batches) | Technical | Crash exactly when a long doc is scanned | Process at sane resolution, compress page JPEGs, recycle bitmaps, stream PDF assembly; P8 100-page test |
| GMS-only scanner shipped to AppGallery/RuStore | Technical/distribution | Scanner silently absent on non-GMS devices | Flavor with an OpenCV implementation for non-GMS stores |
| Interstitial before capture/export | Review-bomb | Task-blocking churn | Post-export, capped placement only; verify in P8 |
| Bundled native `.so` (OpenCV) without 16 KB page-size support | Technical/Play | Fails Play's 16 KB requirement on new devices | Verify all native libs in P3/P4 per `02-engineering/04-sdk-inventory.md` |
| Cloned-template signals (stock UI, boilerplate listing matching thousands of clones) | Play policy (spam/min functionality) | De-indexing or rejection as repetitive content | Differentiated UI in P6, original screenshots/copy in P7 |
| Shadow/glare not handled (blotchy B/W scans) | Quality/reviews | "Scans look bad" 1★ | Background normalization before binarization; use the API's filters; P8 quality corpus |

## Recommended Workflow

1. **P1 Discovery** — classify the existing app's scanner stack (Play-services API vs OpenCV vs a
   clone CV blob), its OCR path (on-device vs cloud), permission set (any storage/contacts/location
   over-ask is an immediate trust/finding flag), and the watermark/paywall model. Note GMS
   dependency vs target stores.
2. **P3 Engineering Audit** — SDK/permission audit with the CamScanner lesson front of mind (vet
   every ad/analytics SDK); OCR privacy posture (does any document leave the device, and is it
   declared?); license + 16 KB compliance of any native CV libs; memory posture on large batches;
   SAF vs all-files storage path.
3. **P4 Modernization** — adopt the Play-services Document Scanner API as the capture/dewarp/filter
   core (drop bespoke clone CV), wire on-device ML Kit OCR for searchable PDF + scan-to-text,
   `PdfDocument` assembly, SAF export; add an OpenCV flavor only if shipping to non-GMS stores.
4. **P5/P6 Design & Redesign** — the capture→batch→review/reorder→export flow and the document-
   gallery home are the rating drivers; auto-capture and reorder are the differentiators.
5. **P7 ASO** — intent + use-case long-tail keyword map; OCR-as-differentiator if earned; trust
   and output-quality screenshots; align every "no watermark / on device" claim with the Data
   safety form.
6. **P8 Verification & Release** — capture/OCR quality corpus, large-batch memory test, watermark
   disclosure check, no-interstitial-before-export code check, Data-safety/privacy consistency.

Category-specific P8 verification additions (append to
`07-checklists/release-readiness.md` items):

```text
[ ] 100-page batch scans, processes, OCRs, and exports to PDF without OOM/ANR
[ ] OCR quality spot-checked on a mixed-language printed-doc corpus; claims match reality
[ ] Free-export watermark is disclosed BEFORE export and never stamped over content
[ ] No interstitial fires before/at capture or before export (code-level check)
[ ] SAF export + FileProvider share work; no MANAGE_EXTERNAL_STORAGE in the manifest
[ ] Data safety form matches actual data flow (on-device vs any cloud OCR/sync)
[ ] Only CAMERA (+ optional READ_MEDIA_IMAGES) requested — no contacts/location/phone
```

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`. Storage-access sibling:
`06-playbooks/document-reader.md`.
