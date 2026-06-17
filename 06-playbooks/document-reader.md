# Document Reader Playbook

Category playbook for PDF, Office, and eBook reader apps (PDF viewers, all-in-one document
readers, EPUB/MOBI readers, DJVU viewers, and "office suite lite" viewers). Load during P1 per
`06-playbooks/README.md` and keep loaded through release. This category is technically mature and
commercially crowded; winning depends on rendering reliability, file-access compliance done the
right way (SAF, not all-files access), and one or two genuine differentiators (annotations, reflow,
TTS) — not on feature-list breadth.

## Category Snapshot

- **Market reality.** One of the highest-volume utility categories on Google Play. Dominated by
  Adobe Acrobat Reader, Google's built-in PDF viewing (Files by Google + in-app PDF rendering in
  Drive), WPS Office, Xodo, and hundreds of near-identical "PDF Reader 2026" clones. Organic
  installs are still winnable on long-tail format keywords (DJVU, CBZ/CBR, EPUB) and on quality
  signals (rating above 4.4, fast cold-open).
- **User intent.** Overwhelmingly transactional: a file just arrived (email attachment, WhatsApp,
  download) and the user needs it open *now*. Second intent: recurring reading (eBooks, manuals)
  where library features and reading comfort matter. Third intent: light manipulation — fill a
  form, sign, annotate, extract pages.
- **Competition shape.** A few giants with full edit suites, then a long tail of ad-stuffed
  viewers. The clone tail trains users to expect ads but also creates the opening: a reader that
  opens files in under 2 seconds, never crashes on malformed PDFs, and shows no interstitial on
  file-open immediately out-reviews 80% of the category.
- **Retention reality.** Most installs are single-use (open one attachment). Retention comes from
  becoming the default handler for the file type (intent filters + "Always" choice) and from the
  library/recents screen being genuinely useful on second open.
- **Competitive reference set** (study during P1 and `04-aso/workflows/competitor-analysis.md`):

| App | What it proves | What to copy / avoid |
|---|---|---|
| Adobe Acrobat Reader | Brand + full edit suite wins head terms | Copy: annotation UX. Avoid: heavy onboarding, account nags |
| Xodo | Quality reader can build a brand without Adobe's budget | Copy: fast open, clean chrome. Note: moved to subscription, reviews dipped |
| WPS Office | All-formats breadth ranks on many keywords | Copy: format coverage messaging. Avoid: ad density (its top 1-star theme) |
| Files by Google | The "good enough" default for casual opens | Differentiate on annotations, reflow, formats it lacks |
| ReadEra (eBooks) | Ad-free + offline + honest = 4.8★ moat | Copy: trust positioning, reading comfort depth |
| Clone tail ("PDF Reader 2026") | Ads-on-open churns users back out | This is the bar to clear, not the model to follow |

## Engineering Patterns

### Rendering engine selection (the central technical decision)

| Engine | Formats | License | Verdict for a closed-source Play app |
|---|---|---|---|
| AndroidX `pdf` (Jetpack, `androidx.pdf:pdf-viewer-fragment`) | PDF | Apache 2.0 | **Preferred for new builds.** Google-maintained, embeddable viewer fragment, sandboxed rendering; newer API — verify current stability tier and minSdk before committing. |
| `PdfRenderer` (platform, API 21+) | PDF | Platform API | Fine for thumbnails and simple page-image viewing; no text layer, no search, no links. Good fallback layer. |
| Pdfium via wrappers (`barteksc/android-pdf-viewer` lineage, maintained forks like `mhiew/android-pdf-viewer`) | PDF | Apache 2.0 (Pdfium is BSD) | Battle-tested, zoom/scroll out of the box. Check fork maintenance and **16 KB page-size compatibility of the bundled `.so`** before adopting. |
| MuPDF | PDF, XPS, EPUB, CBZ | **AGPL** (or paid commercial license) | **License trap.** AGPL contaminates a closed-source app — shipping it without the commercial license is a legal violation. Do not adopt unless the operator buys the Artifex commercial license. |
| PSPDFKit / Nutrient, Apryse | PDF + annotations + forms | Commercial, per-app fees | Excellent but priced for enterprises; almost never justified for an ad-funded utility. |

Office and eBook formats:

- **Office (DOC/DOCX/XLS/XLSX/PPT/PPTX):** Apache POI (Apache 2.0) technically parses these but is
  heavy on Android (multidex pressure, slow on large files, layout fidelity is poor — POI extracts
  content, it does not render WYSIWYG). Realistic options: (a) render *content* (text + tables)
  in a native layout and label it "view", not "edit"; (b) server-side conversion to PDF — best
  fidelity but adds infra cost, latency, and a privacy disclosure burden (user documents leave the
  device — must be in the Data safety form and privacy policy); (c) skip Office and stay
  PDF/eBook-focused. Decide explicitly in P3; do not promise Office fidelity the engine cannot
  deliver.
- **EPUB:** Readium Mobile (kotlin-toolkit, BSD-3) is the serious choice — pagination, reflow,
  TTS hooks. Lighter alternative: render XHTML chapters in WebView with custom CSS (works, but
  search/pagination is on you). Avoid abandoned EPUB libs; check last commit dates in P3.
- **DJVU/CBZ/CBR:** DjVuLibre wrappers are mostly GPL — same contamination check as MuPDF. CBZ is
  just ZIP of images (trivial, built-in `java.util.zip`); CBR needs an unrar lib — `junrar` is
  usable (check its license terms; the unrar algorithm carries redistribution restrictions).

### Storage access (Play-policy-shaped)

- **Use SAF.** `ACTION_OPEN_DOCUMENT` for picking, `ACTION_OPEN_DOCUMENT_TREE` only if a
  folder-library feature truly needs it. Handle `VIEW` intents with `content://` URIs from other
  apps — this is the highest-volume entry point; never assume a `file://` path.
- Canonical open-and-persist flow:

```kotlin
// Pick: persistable grant must be requested at pick time
val pick = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
    addCategory(Intent.CATEGORY_OPENABLE)
    type = "application/pdf"
    putExtra(Intent.EXTRA_MIME_TYPES,
        arrayOf("application/pdf", "application/epub+zip", "image/vnd.djvu"))
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION or
             Intent.FLAG_GRANT_PERSISTABLE_URI_PERMISSION)
}
// On result: persist for recents, then open via ContentResolver — never File(path)
contentResolver.takePersistableUriPermission(uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
contentResolver.openFileDescriptor(uri, "r")?.use { pfd -> renderer.open(pfd) }
```

- Manifest intent filters make the app a `VIEW` handler (the retention mechanism):

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <data android:scheme="content" android:mimeType="application/pdf" />
</intent-filter>
<!-- Repeat per supported MIME type; add android:scheme="file" for legacy senders. -->
```

- Test the `VIEW` entry path directly during P8:

```powershell
adb shell am start -a android.intent.action.VIEW `
  -d "content://com.android.providers.downloads.documents/document/123" `
  -t "application/pdf"
```
- **Do NOT request `MANAGE_EXTERNAL_STORAGE`.** Play's all-files-access declaration is routinely
  **rejected for document readers** — the policy carve-out targets file managers and antivirus,
  and reviewers consistently rule that a reader's core function works via SAF. Requesting it risks
  rejection and flags the app for scrutiny. `READ_MEDIA_*` permissions don't cover PDFs either;
  SAF is the whole story on API 30+.
- **Recents/library:** call `takePersistableUriPermission` on every opened document and store the
  URI + display name + last page + timestamp in Room. Persisted URI grants are capped per app
  (historically 128, raised to 512 on API 30+) — evict oldest grants LRU when near the limit, and
  gracefully drop recents whose grant or backing file is gone (catch `SecurityException` /
  `FileNotFoundException` on reopen; show "file no longer available", don't crash).

### Architecture sketch (aligns with `08-knowledge/android/modern-stack.md`)

- `:core:rendering` — engine abstraction (`DocumentEngine` interface: open, pageCount,
  renderPage(index, size), searchText, outline). One implementation per engine; swapping engines
  later (license/maintenance reasons) touches one module.
- `:core:library` — Room database of recents/bookmarks/positions keyed by persisted URI string +
  document hash (URI alone breaks when the provider re-issues IDs).
- `:feature:reader` — Compose UI; page bitmaps exposed as a `Flow<PageState>`; chrome state via a
  single `ReaderUiState`.
- DI via Hilt; engine choice injected so a debug build can A/B engines on the same corpus.

### Performance discipline (large files are the norm)

- Render **page-level, never whole-document**: keep a window of current ±1–2 pages as bitmaps,
  recycle aggressively, render at view resolution (not intrinsic page size), re-render tiles on
  zoom rather than decoding at max zoom upfront.
- Open the document **off the main thread** (coroutine + `Dispatchers.IO`), show page 1 fast,
  lazy-build the page list/thumbnails. Cold-open-to-first-page is the category's headline metric;
  budget < 2 s for a typical 5 MB PDF.
- **Malformed files crash readers.** Wrap open/parse in defensive try/catch, handle
  password-protected PDFs with a password dialog (not a crash), and test against a corpus of
  corrupt/truncated/encrypted PDFs in P8 (`01-core/prompts/15-final-verification.md`).

### Differentiators worth building (in ROI order)

1. **Search in document** — table stakes for "reader" reviews; requires a text layer (AndroidX pdf
   or Pdfium text APIs).
2. **Annotations** (highlight, freehand, sticky note) — the #1 pro-tier feature in the category.
3. **Text reflow / reading mode** for PDFs on phones — rare, loved, technically hard (text
   extraction + re-layout).
4. **TTS read-aloud** — platform `TextToSpeech`, cheap to add over a text layer, strong
   accessibility + ASO angle.

## Design Patterns

- **Home = library/recents, not a file picker.** First screen shows recent documents as a list or
  grid with thumbnails, format badges, and resume-position indicators; a prominent "Open file"
  action launches SAF; optional tabs by format (PDF / Word / EPUB). Top apps (Acrobat, Xodo, WPS)
  all converge on this. If the app opens via a `VIEW` intent, skip home and go straight to the
  reader.
- **Reader chrome: immersive, tap-to-reveal.** Content full-bleed and edge-to-edge (mandatory at
  targetSdk 35+); single tap toggles top bar (title, search, more) and bottom bar (page slider,
  page x/y). Auto-hide after ~3 s. Pinch-zoom + double-tap zoom are assumed; missing them reads
  as broken.
- **Reading comfort controls:** night mode (true content inversion for PDFs — render-time color
  transform, not just dark chrome; preserve embedded images un-inverted where the engine allows),
  sepia, brightness slider, keep-screen-on toggle, horizontal page-flip vs vertical continuous
  scroll choice. eBook mode adds font size/family, line spacing, and margins. Persist all of it
  per mode in DataStore; comfort settings that reset are a recurring complaint.
- **Always resume at last position.** Per document, persisted, instant. Its absence is a recurring
  1-star theme in the category. Key it to a content hash, not just the URI, so the same file
  re-shared from a different provider still resumes.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Library / Recents | Re-entry value; default-app payoff | Thumbnails, resume badges, format filter, "Open file" CTA |
| Reader | The product | Immersive, edge-to-edge, tap-to-reveal chrome |
| Search-in-document | Table-stakes utility | Overlay field, hit count, prev/next, highlight on page |
| Outline / Bookmarks drawer | Long-document navigation | TOC from engine outline API; user bookmarks per URI |
| Annotation toolbar | Pro-tier value | Highlight, ink, note; undo; export flattened copy |
| Settings | Reading prefs | Night/sepia, scroll direction, keep-screen-on, default-viewer how-to |

- **TTS read-aloud:** play/pause in reader chrome, sentence highlight tracking, background
  playback with a media notification. It is simultaneously an accessibility win (document it in
  `03-design/09-accessibility-review.md` output) and an ASO bullet.
- **Accessibility floor for the category:** content descriptions on all chrome icons, page
  announcements for TalkBack on page change, minimum 48 dp touch targets on the page slider —
  reader apps fail review disproportionately on the slider.

## ASO Patterns

- **Keyword themes by intent tier** (build the full map in P7 via
  `04-aso/workflows/keyword-research.md`):

| Tier | Examples | Volume | Competition | Role |
|---|---|---|---|---|
| Head | "pdf reader", "pdf viewer", "pdf" | Very high | Brutal (Adobe, WPS) | Title presence; rank is a months-long velocity project |
| Category | "document reader", "ebook reader", "office viewer" | High | High | Short description + long description coverage |
| Task | "open pdf file", "pdf with search", "read aloud pdf" | Medium | Medium | Long-description sentences in natural phrasing |
| Format long-tail | "djvu reader", "cbz reader", "mobi reader", "xps viewer" | Low each, additive | Weak | **The realistic ranking entry point for a new/modernized app** |

- **Localization multiplies the long tail cheaply, but format terms have per-locale gotchas.**
  Prioritize the highest-volume store locales for this category: **en, es-419 (Latin-American
  Spanish — distinct from es-ES), pt-BR, hi (India is a top install market for free readers),
  id, ru, de, fr.** Run `04-aso/workflows/localization.md` for these. Per-locale format-term
  traps that machine translation gets wrong: "PDF" stays Latin everywhere (don't transliterate
  in ru/hi); German compounds the noun — "PDF-Reader", "PDF-Leser", and "Dokument-Viewer" are
  three different head terms, pick by volume not by literal translation; Spanish users search
  "lector de PDF" and "abrir PDF", not the English-order "PDF lector"; Portuguese "leitor de PDF"
  / "abrir arquivo PDF"; Russian uses both "читалка PDF" (colloquial) and "просмотр PDF"
  (formal) — seed both; Hindi searches are frequently Romanized English ("pdf reader") even in
  the hi locale, so keep English seeds in the hi candidate set. EPUB/DJVU/CBZ stay Latin
  acronyms across locales (an easy, safe long-tail multiplier).
- **Title formula:** `<Brand>: PDF Reader & Viewer` or `PDF Reader — <differentiator>` (e.g.
  "PDF Reader — Fast & No Ads in Reader"). Lead with the category keyword within Play's 30-char
  title; put format breadth ("EPUB, DJVU, Word") in the short description.
- **Screenshot messaging that converts:** screenshot 1 = a real document rendered crisply with a
  speed/trust caption ("Opens any PDF instantly"); 2 = night/comfort modes; 3 = annotations or
  search; 4 = all-formats grid; 5 = library screen. **Trust signals matter disproportionately**
  here (users open sensitive documents): captions like "Works offline — files never leave your
  device" convert, and must be true (verify against the Data safety form in P8).
- Mirror claims with the listing: if the listing says "no ads while reading", that becomes a
  product requirement enforced in Monetization Patterns below.
- **Ratings are a ranking input here more than in most categories** because head-term SERPs show
  several near-identical titles — the star delta decides the tap. Protect the rating with the
  ad rules below and use in-app review prompts (`ReviewManager`) only after a successful
  multi-open week, never after the first file.

- **Category benchmark (est.):** store-listing **CVR ~22–32%** for a clean reader on transactional
  head/format terms (high-intent traffic converts well; "open this file now" is a strong buyer
  signal). **Rating norm ~4.3–4.6** — the giants and the trust-positioned readers (ReadEra) sit
  at the top, the ad-stuffed clone tail drags the median down. Treat anything under 4.2 as a
  rating-recovery project before ASO spend. Use these as the baseline in the ASO_HANDOVER
  Baseline Metrics Snapshot (`_V11_SPEC` D16) and to set CVR/rating goals in Prompt 12.

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1):**

```text
pdf reader, pdf viewer, pdf, document reader, read pdf, open pdf file,
pdf reader for android, ebook reader, epub reader, office viewer, doc reader,
word document reader, pdf with search, search in pdf, read aloud pdf,
pdf reader no ads, pdf reader offline, all document reader, file reader,
djvu reader, cbz reader, cbr reader, mobi reader, xps viewer, txt reader,
pdf annotator, sign pdf, pdf night mode, fast pdf reader, book reader
```

- **Review-reply templates for the "can't open my file" 1★ class.** The dominant negative review
  in this category is "won't open my file / crashes / blank page" — usually a malformed,
  encrypted, or unsupported-format document, not a general defect, and frequently recoverable
  with a reply. Replying converts a meaningful share of these to edited ratings and reads as a
  trust signal to listing visitors. Triage and reply per the canonical workflow in
  `04-aso/workflows/ratings-reviews.md` (response cadence, escalation, and tone rules live there;
  these are the category-specific starter strings):

| 1★ symptom | Likely cause | Reply template (adapt, don't paste robotically) |
|---|---|---|
| "Won't open my PDF / blank page" | Malformed or non-standard PDF | "Sorry that file wouldn't open. Some PDFs are exported in non-standard ways — could you tap Menu → Send feedback and attach it (or tell us the app it came from)? We fix these fast and will reply when it's handled." |
| "Asks for a password / can't read it" | Encrypted PDF | "That file is password-protected by whoever sent it — enter the password on the unlock prompt and it'll open. If there's no prompt, update to the latest version; we added the unlock dialog in vX.Y." |
| "Doesn't open my .epub/.djvu/.cbz" | Unsupported or mislabeled format | "We do support that format — if it still won't open it may be misnamed or corrupt. Send it via Menu → Send feedback and we'll check the specific file." |
| "Crashes on big files" | Memory/large-PDF handling | "Thanks for flagging — large files should open page-by-page without crashing. We've improved this in vX.Y; please update and let us know if it persists." |
| "Lost my reading position after update" | Migration regression | "That shouldn't happen and we're sorry — resume-position is meant to survive updates. We're fixing the migration in the next release; reply here and we'll confirm when it ships." |

  Template the final strings into `docs/factory/assets/` during P7 and wire the
  Review-Response Readiness section of the ASO_HANDOVER (`_V11_SPEC` D16) to them.

## Monetization Patterns

- **Placement matrix:**

| Placement | Verdict | Rationale |
|---|---|---|
| Banner/native in library list | Allowed | User is browsing, not mid-task |
| App-open ad on cold start | Allowed, capped 1/session | Never on `VIEW`-intent entry — user has a task |
| Interstitial on file-open | **Forbidden** | Review-bomb cause #1; kills default-handler loop |
| Interstitial on reader-exit | Allowed, capped | 1 per 3 exits, ≥ 120 s spacing |
| Any ad over document content | **Forbidden** | Mis-taps = accidental-click policy + rating damage |
| Rewarded ad for one-off pro action | Allowed | E.g. one annotated export without pro |

- App-open ad cap and `VIEW`-intent suppression must be implemented in code (entry-point check),
  not left to ad-network defaults; verify in P8.
- **THE rule: no interstitial on file-open.** Interstitial-before-document is the category's
  review-bomb cause #1 ("I just wanted to open my ticket and got a full-screen ad"). It also
  tanks the default-handler retention loop. Never place it there regardless of eCPM pressure.
- **No ads inside the reading surface.** Banners over document content cause mis-taps, policy risk
  (accidental-click layouts), and rating damage.
- **Free/pro split:** free = view all formats, search, recents, night mode, with ads. Pro
  (one-time purchase converts better than subscription in this category; offer both if testing) =
  no ads + annotations + page edit (extract/rotate/reorder) + themes. Paywall placement: on tap of
  a pro feature (contextual), never blocking file viewing.
- Detail patterns live in `08-knowledge/monetization/ads-patterns.md` and
  `subscription-patterns.md`; this section overrides them where they conflict for this category.

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| MuPDF/DjVuLibre (AGPL/GPL) linked into a closed-source app | Legal/license | Takedown exposure, forced open-sourcing claims | License audit in P3 (`02-engineering/03-dependency-analysis.md`); only Apache/BSD/MIT engines, or buy commercial license |
| Requesting `MANAGE_EXTERNAL_STORAGE` | Play policy | Declaration rejected; update blocked | SAF-only architecture; remove the permission and any code path needing it |
| Interstitial on file-open | Review-bomb | 1-star avalanche, uninstalls, default-handler loss | Hard rule above; verify in P8 ad-placement pass |
| Crash on malformed/encrypted PDF | Technical | Crash-rate ANR/quality flags + bad reviews at the worst moment | Defensive parsing, password dialog, corrupt-file test corpus in P8 |
| Cloned-content / template-app signals (stock UI, boilerplate listing identical to thousands of clones) | Play policy (spam & minimum functionality) | Rejection or de-indexing as low-quality/repetitive content | Differentiated UI in P6, original screenshots/copy in P7; never reuse clone-template metadata |
| Persisted-URI grant limit overflow or stale grants | Technical | Recents silently break / crash on reopen | LRU grant eviction; catch and prune dead entries |
| Office "editing" promises with a view-only engine | Honesty/reviews | Misleading-claims complaints, refunds | Label Office support as "view"; align listing copy with real capability in P7 |
| Bundled native `.so` without 16 KB page-size support | Technical/Play | Fails Play's 16 KB requirement on new devices | Verify all native rendering libs in P3/P4 per `02-engineering/04-sdk-inventory.md` |
| Losing reading position / comfort settings across updates | UX/reviews | "Update ruined it" review wave | Room/DataStore schema migrations tested in P8; never wipe on upgrade |
| Abandoned engine fork (no maintainer, no 16 KB/edge-to-edge fixes) | Technical debt | Forced emergency migration later | Check fork commit recency in P3; keep `DocumentEngine` abstraction so swaps stay cheap |

## Recommended Workflow

1. **P1 Discovery** — classify formats actually supported vs claimed; identify the rendering
   engine and its license immediately (an AGPL finding changes the whole roadmap).
2. **P3 Engineering Audit** — heavy emphasis: license audit of every parsing/rendering dependency;
   storage-access audit (any `MANAGE_EXTERNAL_STORAGE`/legacy `READ_EXTERNAL_STORAGE` reliance →
   SAF migration becomes a Critical roadmap item); crash posture on malformed files; native-lib
   16 KB compliance.
3. **P4 Modernization** — engine migration (if license- or maintenance-forced) before anything
   cosmetic; SAF + persisted-URI recents pipeline; page-level rendering memory discipline.
4. **P5/P6 Design & Redesign** — library-home + immersive reader chrome per Design Patterns;
   night/sepia modes; resume-position. These are the rating drivers.
5. **P7 ASO** — long-tail format keyword map first; trust-signal screenshots; align every listing
   claim with verified capability.
6. **P8 Verification & Release** — corrupt-file corpus test, `VIEW`-intent matrix test
   (Gmail/WhatsApp/Chrome/Files sources), ad-placement rule check (no interstitial on open), Data
   safety form consistency with "offline/private" claims.

Category-specific P8 verification additions (append to
`07-checklists/release-readiness.md` items):

```text
[ ] Opens 50-file corpus (incl. corrupt, encrypted, 200MB+, 1000-page) without crash/ANR
[ ] Cold-open to first page < 2 s on a mid-range device for 5 MB PDF
[ ] VIEW intent from Gmail, WhatsApp, Chrome, Files all resolve and render
[ ] No interstitial fires on file-open or VIEW-intent entry (code-level check)
[ ] Recents survive reboot; dead URIs pruned without crash
[ ] No AGPL/GPL artifact in the final license report (./gradlew licenseReport or equivalent)
```

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`.
