# Data Recovery Playbook

Category playbook for photo/video/file recovery apps ("recover deleted photos", undelete tools,
recycle-bin apps). Load during P1 per `06-playbooks/README.md` and keep loaded through release.

> **HONESTY-CRITICAL CATEGORY — read this first.** On modern non-rooted Android, **true undelete
> is mostly impossible.** Internal storage is encrypted (file-based encryption) and SSDs issue
> TRIM, so deleted file blocks are genuinely gone within minutes; no app without root can carve
> them back. What IS recoverable: media still indexed as trashed in MediaStore (Android 11+
> trash, 30-day window), `.trashed-*` files and app-side recycle bins, app caches and thumbnails
> (degraded copies), cloud-synced duplicates (Google Photos trash holds 60 days), and raw-scan
> remnants on **removable SD cards** (typically FAT32/exFAT, no TRIM guarantee, accessible with
> all-files or SAF tree access). Play's misleading-claims enforcement is ACTIVE in this category
> — apps promising what they cannot do get suspended, and distressed users review-bomb them
> first. The only durable strategy: **build honest recovery (trash + thumbnail/cache scan +
> cloud-restore guidance) plus PREVENTION positioning (a recycle bin going forward).** Every
> section below assumes that spine.

## Category Snapshot

- **Market reality.** Massive emotional-intent search volume ("recover deleted photos" and
  variants) chased by a graveyard of suspended apps. The surviving field splits into: (a)
  honest trash/duplicate utilities, (b) PC-tethered tools (the actual deep-recovery products,
  which run from a computer), and (c) a churn of overpromising apps that monetize desperation
  until enforcement or review-bombs kill them. The honest niche is under-served and defensible.
- **User intent.** Acute distress: a specific photo/video/folder was just deleted — wedding
  photos, a deceased relative's pictures, a work document. The user is emotional, impatient, and
  willing to pay *if shown the actual item*. Secondary intent (calmer, retainable): "make sure
  this never happens again" — the prevention/recycle-bin intent.
- **Competition shape.** Competitors compete on promise size, not capability — which is exactly
  why ratings in the category are bimodal (5-star "found my photos!" from trash recoveries,
  1-star "scam, showed nothing/asked money first"). An app that sets expectations correctly and
  recovers what is genuinely recoverable collects the 5-star half without the suspension risk.
- **Strategic posture:** recovery is the acquisition hook; the recycle-bin/prevention feature is
  the retention product. Plan both from P1.
- **Competitive reference set** (study during P1 and `04-aso/workflows/competitor-analysis.md`):

| Competitor type | Example behavior | Lesson |
|---|---|---|
| Overpromiser (rotating clones) | "Deep scan recovers everything!", pay-before-reveal | Their suspensions free up rank; never copy their claims |
| Honest trash utility | Restores system trash, duplicate cleanup, modest claims | The defensible model; differentiate on triage UX + protection |
| PC-tethered tools (DiskDigger-class with root, desktop suites) | Real deep recovery, root or desktop required | Acknowledge in cloud/edge cases; don't pretend to match them rootless |
| Gallery-vault / backup apps | Prevention-first positioning | The adjacency the protection product competes in |

## Engineering Patterns

### What is actually recoverable (build to this list, promise only this list)

| Source | Mechanism | Quality | Window |
|---|---|---|---|
| MediaStore trash (Android 11+) | `MediaStore` query with `QUERY_ARG_MATCH_TRASHED` / `IS_TRASHED=1`; restore via `createTrashRequest(resolver, uris, false)` (user-consented system dialog) | Full original | ~30 days from deletion |
| App-side recycle bins (gallery vendors, file managers) | Scan known `.trashed-*` / vendor bin paths (needs all-files access or SAF tree) | Full original | Vendor-defined |
| Thumbnails & app caches | Scan `.thumbnails`, messenger media caches, `Glide`/`Coil` disk caches | **Degraded** (low-res copies) | Until cache eviction |
| Cloud-synced copies | Guidance flows: Google Photos trash (60 days), Drive/OneDrive/Dropbox trash, device-backup restore steps | Full original | Service-defined |
| Removable SD card raw scan | Read block/free-space carving for JPEG/MP4/PNG signatures on FAT32/exFAT; needs SAF tree or all-files access; effectiveness drops with card usage since deletion | Variable (fragmentation breaks files) | Until overwritten |
| Internal storage raw scan, no root | — | **Not possible** (FBE encryption + TRIM). Never imply otherwise. | — |
| Rooted-device deep scan | Block-level carving with root | Variable | Tiny audience; separate risk profile — out of scope for Play builds |

- **MediaStore trash APIs are the workhorse:** query trashed items with
  `MediaStore.QUERY_ARG_MATCH_TRASHED`, surface them with thumbnails, restore via
  `MediaStore.createTrashRequest(...)` — the system consent dialog handles permission per batch.
  Pair with `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` (33+) or `READ_EXTERNAL_STORAGE` (≤32).

```kotlin
// Query trashed media (API 30+)
val args = Bundle().apply {
    putInt(MediaStore.QUERY_ARG_MATCH_TRASHED, MediaStore.MATCH_ONLY)
}
resolver.query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    arrayOf(MediaStore.Images.Media._ID,
            MediaStore.Images.Media.DISPLAY_NAME,
            MediaStore.Images.Media.DATE_EXPIRES), args, null)

// Restore: system consent dialog, batched
val request = MediaStore.createTrashRequest(resolver, selectedUris, /* trash = */ false)
trashLauncher.launch(IntentSenderRequest.Builder(request.intentSender).build())
```

- Surface `DATE_EXPIRES` per item ("recoverable for 12 more days") — it is true urgency, the
  honest version of the countdown timers dark-pattern apps fake.
- **SD raw scan feasibility:** signature carving over the card's readable space is real but:
  requires broad read access (all-files declaration for a recovery tool is *arguable* but weaker
  than a file manager's case — prepare a SAF-tree fallback and honest messaging if the
  declaration is rejected), is slow (stream, don't load; show real progress per region scanned),
  and yields fragmented/partial files — mark partials clearly. Signature set worth implementing:

| Format | Header signature | Trailer / structure | Carving notes |
|---|---|---|---|
| JPEG | `FF D8 FF` | `FF D9` | Most common target; thumbnails embedded in EXIF can double-count |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `IEND` chunk | Chunk lengths allow integrity validation |
| MP4/MOV | `ftyp` atom at offset 4 | Atom-walk to length | Fragmentation usually breaks video; mark Partial |
| HEIC | `ftypheic`/`ftypmif1` | Atom-walk | Default camera format on many devices since Android 9 era |
| GIF/WEBP | `GIF8` / `RIFF....WEBP` | Format-defined | Low value; include for completeness |
- **Recycle-bin-forward architecture (the prevention product):** Android offers no global
  delete-intercept hook for third-party apps — be straight about the limits. Viable mechanisms:
  (a) the app's own gallery/file UI where deletes go to an app-managed bin (works only for
  deletions done *inside the app*), (b) periodic snapshot/sync of selected folders to an
  app-private or user-chosen backup location (WorkManager periodic job — this is genuinely
  protective and the honest pitch), (c) `MediaStore.createTrashRequest` integration so in-app
  deletes use the system 30-day trash, (d) monitoring MediaStore deltas to *notice* deletions and
  prompt "X items were deleted — restore from your <app> backup?" Do not market (d) as
  interception; market the backup as the protection.
- **Scan engine discipline:** coroutine-based scanner with cancellable, region-reporting
  progress; Room index of findings (source, path/URI, type, quality tier, recoverability);
  thumbnail generation throttled to avoid OOM on 10k-result scans.
- **Protection product job spec (the subscription's substance):**
  1. User picks folders to protect (SAF trees or media collections) and a destination
     (app-private dir, user SAF tree, or their cloud drive folder).
  2. `PeriodicWorkRequest` (e.g. every 6 h, `BatteryNotLow` + storage constraints) diffs the
     protected set against the backup index and copies new/changed items.
  3. MediaStore delta check flags disappeared items → notification: "3 photos were deleted —
     they're safe in your <App> vault."
  4. Vault browser with restore-to-original-album; retention window user-configurable.
  This loop is honest, testable, and the reason the subscription survives review scrutiny.
- **Libraries:** all platform APIs plus standard stack (Room, WorkManager, Coil — all
  Apache 2.0). This category needs no exotic dependencies; treat any "recovery SDK" bundled in a
  legacy target app as a P3 red flag and audit what it actually does (several legacy "recovery
  engines" are obfuscated ad/analytics bundles wearing a scanner costume).

## Design Patterns

- **Empathy tone throughout.** The user is distressed. Copy is calm, concrete, step-respecting:
  "Let's check the places deleted photos can still be" — never casino-style "DEEP SCAN NOW!!"
  Avoid red alarm UI; use steady progress and plain language.
- **Honest scan-progress theater line:** progress must reflect **real work** (sources checked,
  regions scanned, items found — show the counts live). Fake progress bars and inflated
  "Scanning 98,432 files…" animations over a no-op are (a) detectable by reviewers, (b) a
  misleading-functionality signal, (c) the thing 1-star reviews quote verbatim. A fast real scan
  beats slow fake drama.
- **Results triage UX — the clarity contract:** group results by recoverability, visibly:
  **"Recoverable — full quality"** (trash items), **"Preview only — reduced quality"**
  (thumbnails/cache copies, labeled with resolution), **"Partial"** (raw-scan fragments), and a
  **"Check your cloud"** guidance card (Google Photos trash deep-link + steps). **Never present
  an unrecoverable or degraded item as recoverable** — mislabeling here is the category's core
  sin and the root of refund storms.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Home | Two clear paths | "Recover deleted files" / "Protect my files" — equal visual weight |
| Scan progress | Trust via visible real work | Live source names + found counts; cancellable |
| Results triage | The honesty surface AND the conversion surface | Recoverability groups, quality labels, `DATE_EXPIRES` countdowns |
| Preview | Informed decision | Full-screen, resolution/quality label, no watermark tricks |
| Restore confirmation | Closure | Show exact destination album/path; open-in-gallery action |
| Protection setup | The retention product | Folder picks, schedule, storage location, status card on home |
| Cloud-restore guides | Honest breadth | Illustrated steps per service + deep links where possible |

- **Copy bank discipline:** maintain reviewed strings for the hard moments — "We couldn't find
  this photo. Here's why that happens on modern phones, and what may still work" with the cloud
  guide linked. Honest dead-ends, handled warmly, generate the category's rare grateful reviews.
- **First-run matters more than usual:** the user's patience is measured in seconds. Sequence:
  1. One screen: "What were you trying to recover?" (Photos / Videos / Other) — routes the scan
     and personalizes copy.
  2. Request the minimum permission for that route with a one-line plain-language reason.
  3. Start the scan immediately on grant; no account creation, no feature tour, no
     notification-permission prompt until after first results.
  4. After first restore (or honest dead-end), offer the protection setup — this is the moment
     of maximum receptiveness.

## ASO Patterns

- **Keyword themes by intent tier (enormous emotional volume):**

| Tier | Examples | Notes |
|---|---|---|
| Emotional head | "recover deleted photos", "photo recovery", "restore deleted pictures" | Highest volume; competitors churn out via suspension — longevity wins rank |
| Media variants | "recover deleted videos", "deleted video recovery" | Same intent, less contested |
| Prevention | "recycle bin", "trash bin for android", "photo backup" | The retention product's keywords; calmer intent, better LTV |
| Situational long-tail | "recover photos from sd card", "restore whatsapp images" | Only claim what the app truly covers |
- **Claims must match reality — this is the enforced category.** Play's misleading-claims
  enforcement actively targets recovery apps. Rules for ALL metadata: never promise recovery of
  internal-storage files deleted outside trash windows; qualify scope ("recover photos from
  trash, caches and SD card"); no fake before/after miracles. The long description should plainly
  state what is and isn't possible — it filters refunders, pre-empts 1-stars, and reads as
  credibility next to overpromising competitors.
- **Screenshot honesty rules:** show the real results-triage screen with the real quality labels;
  no fabricated "1,247 photos recovered!" counters; no implication of recovering
  years-old internal-storage deletions. Screenshot 1 = results triage with found items ("Finds
  photos that can still be saved"); 2 = trash restore flow; 3 = protection/recycle-bin setup
  ("Never lose a photo again"); 4 = cloud-restore guidance; 5 = SD card scan.
- **Title formula:** `Photo Recovery: Trash & Backup` / `<Brand> — Recover Photos & Recycle Bin`.
  Pair the recovery keyword with the prevention keyword — it widens intent coverage and signals
  the honest positioning to reviewers.
- **Review-reply strategy is ASO here:** distressed 1-stars ("didn't find my photos") answered
  with a concrete, kind explanation + cloud-restore pointer visibly convert to edited ratings and
  read as trust signals to listing visitors. Budget daily reply time for the first month
  post-release; template the replies in `docs/factory/assets/` during P7 and wire them to the
  ASO_HANDOVER Review-Response Readiness section (`_V11_SPEC` D16). Triage cadence, escalation
  paths, and the rule for reporting competitor review-brigades all live in the canonical
  `04-aso/workflows/ratings-reviews.md`; this category just needs that workflow run daily and
  warmly, because the 1-stars here arrive at maximum emotional stakes.

- **Category benchmark (est.):** store-listing **CVR ~20–30%** — emotional intent converts hard
  ("recover deleted photos" is among the highest-intent utility queries), which is exactly why
  overpromisers exploit it; an honest listing converts toward the middle of the band but holds
  the install instead of refunding it. **Rating norm is bimodal, ~3.5–4.2 effective** — the
  category's structural 5★/1★ split (trash recoveries vs. "scam, found nothing") pulls medians
  down, so an honest app that clears the pay-before-reveal trap can credibly target the top of
  the band where the overpromisers cannot follow. Record actuals against this in the
  ASO_HANDOVER Baseline Metrics Snapshot (`_V11_SPEC` D16).

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1 — claim
  only what the app truly covers per the recoverability table; do NOT seed
  internal-storage/"deep recovery" promises the rootless app can't keep):**

```text
recover deleted photos, photo recovery, restore deleted photos, deleted photo recovery,
recover deleted videos, deleted video recovery, restore pictures, file recovery,
recover deleted files, undelete photos, photo restore, recover photos from sd card,
recover photos from trash, restore from recycle bin, recycle bin, trash bin,
photo backup, recover lost photos, restore deleted images, get deleted photos back,
data recovery, image recovery, recover deleted media, photo vault backup
```

## Monetization Patterns

- **The category's dark pattern — documented so it is never replicated:** scan free → show a
  grid of "recoverable" items (often unrecoverable thumbnails presented as full photos) →
  **paywall before any item can be viewed or restored** ("pay $9.99 to recover"). Why it
  review-bombs: the user pays under duress, receives a low-res thumbnail or nothing, feels
  scammed at maximum emotional stakes, leaves a 1-star with the word "scam", files a refund, and
  sometimes a Play policy complaint. This pattern is also what triggers misleading-functionality
  suspensions. **Do not build pay-before-reveal.**
- **The ethical alternative that still converts:** scan free, **preview free** (full-screen,
  honest quality label), restore-from-trash free or freemium-limited (e.g. N items/day free) —
  **pay to export full-resolution in batch, unlock SD deep scan, and unlock the protection
  product** (automatic backup/recycle bin, ongoing value = honest subscription justification).
  The user pays after seeing exactly what they'll get; refunds and rage collapse.
- **Ads:** light touch only — the user is distressed and ad-blasting them converts to uninstalls.
  Acceptable: banner/native on results and home; rewarded ad as an alternate unlock for a single
  restore. **Never an interstitial between scan and results** (it reads as withholding their
  photos for ad money) and never during scan progress. Disable app-open ad formats on any route
  that begins a recovery flow.
- **Free/pro split summary:**

| Capability | Free | Pro |
|---|---|---|
| Scan all sources | Yes | Yes |
| Full-screen preview with quality label | Yes | Yes |
| Restore from system trash | Yes (or N/day) | Unlimited |
| Batch / full-resolution export | — | Yes |
| SD card raw scan | — | Yes |
| Automatic protection (backup + bin) | Trial | Yes (the subscription justification) |
| Ads | Light | None |

- **Pricing posture:** one-time "rescue" IAP for the one-off rescuer + monthly/annual
  subscription for protection. Never price-gate by desperation: no surge-style "limited time to
  recover" pressure, and **no fake offer countdowns**. The only legitimate countdown in this app
  is a *real, fixed offer expiry* — i.e. a genuine introductory-price window that ends at the
  same real date for every user and does not silently reset when they reopen the app (the system
  `DATE_EXPIRES` trash-window countdown surfaced in the results UI is real urgency and fine; a
  paywall timer that restarts on each launch is a dark pattern and a misleading-claims signal).

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| Misleading-functionality suspension (promising impossible undelete) | Play policy — **the #1 killer in this category** | App suspension; strikes toward account ban | Honest-scope metadata and UI per ASO Patterns; P7 claim-by-claim audit against the recoverability table |
| Fake-scan detection by Play reviewers (progress theater over no-op, fabricated result counts) | Play policy | Suspension; deceptive-behavior flag | Real, observable scan work; progress tied to actual sources/regions |
| Pay-before-reveal paywall | Reviews/refunds/policy | Review-bomb, refund storm, policy complaints | Free-preview/pay-to-export model above |
| Showing unrecoverable or thumbnail-only items as "recoverable" | Honesty/UX | "Scam" reviews, refunds | Triage labels (full / preview-only / partial) enforced in the results UI |
| Refund storms from distressed buyers | Business | Revenue clawback + Play account health signals | Preview-first monetization; clear pre-purchase expectation copy |
| Competitor review-bombs (this category attracts them) and organic 1-star brigades | Reputation | Rating sink | Monitor reviews daily post-release; respond with concrete help; report off-topic brigades via Play Console |
| All-files-access declaration rejected for the SD-scan feature | Play policy | Feature blocked | SAF-tree fallback path built from day one; never make SD scan the only headline feature |
| OOM/ANR on 10k-result scans | Technical | Crashes at the moment of maximum user stress | Streamed scanning, paged results, throttled thumbnailing; P8 load test with a full SD card |
| Restoring into the wrong album / silent restore destination | UX/trust | "It said restored but I can't find it" 1-stars | Restore confirmation screen shows destination; open-in-gallery action |
| Ad SDK loading interstitials during scan via automatic triggers | Reviews/policy | Reads as ransom of the user's photos | Disable automatic/app-open formats on scan and results routes in code |
| Protection backups silently stopping (OEM battery killers) | Trust | Product fails exactly when needed | Status card with last-run timestamp; re-schedule checks; vendor-whitelist guidance screen |
| Marketing delete-interception the OS doesn't allow | Honesty | Broken promise discovered in week one | Position prevention as backup + system trash integration, per Engineering Patterns |

## Recommended Workflow

1. **P1 Discovery (heavy, honesty-gated)** — inventory what the existing target app *claims* vs
   what its code *does* (decompile-level honesty audit: does the scan actually read anything?).
   Map every claim to the recoverability table. Output a claims-vs-capability matrix in
   `docs/factory/audits/DISCOVERY.md`; any impossible claim becomes a Critical P7 fix and a
   possible Critical product change. If the existing app's monetization is pay-before-reveal,
   flag the revenue impact of removing it to the operator explicitly — with the suspension and
   refund data that justifies the change — and record the decision in
   `docs/factory/PROJECT_STATE.md`.
2. **P3 Engineering Audit** — scan-engine reality check, permission posture, monetization-flow
   audit (find any pay-before-reveal gate — Critical finding), third-party "recovery SDK"
   inspection.
3. **P4 Modernization** — MediaStore trash integration, real scan engine with honest progress,
   protection/backup product (WorkManager), results triage data model.
4. **P5/P6 Design & Redesign (elevated)** — empathy tone, triage clarity, preview-first flows;
   in this category the design IS the honesty mechanism.
5. **P7 ASO (elevated, claims-led)** — claim-by-claim metadata audit against capability BEFORE
   keyword work; then emotional-intent keyword map and honest screenshots; rewrite long
   description as the expectation-setting surface.
6. **P8 Verification & Release** — recoverability-label correctness tests (no unrecoverable item
   shown as recoverable), monetization-flow walkthrough (preview reachable without payment),
   load-test scans, staged rollout with daily review monitoring and refund-rate tracking as
   rollback triggers.

Category-specific P8 verification additions (append to
`07-checklists/release-readiness.md` items):

```text
[ ] Every results-screen label verified against actual source (trash item != cache thumbnail)
[ ] Preview reachable for every found item with zero payment and zero rewarded ad
[ ] Scan progress halts/cancels cleanly; counts shown match Room index contents
[ ] DATE_EXPIRES countdowns match MediaStore values on a seeded test device
[ ] 10,000-item scan on a full 128 GB SD card: no OOM, results paged
[ ] Listing claims cross-checked line-by-line against the recoverability table (P7 artifact)
[ ] Refund-rate and 1-star-rate dashboards wired before rollout exceeds 10%
```

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`. Monetization baseline:
`08-knowledge/monetization/subscription-patterns.md`.
