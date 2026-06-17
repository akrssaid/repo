# Video Downloader Playbook

Category playbook for video/media downloader apps (social-video savers, m3u8/HLS downloaders,
generic URL media grabbers, "story saver" variants). Load during P1 per `06-playbooks/README.md`
and keep loaded through release.

> **POLICY-CRITICAL CATEGORY — read this first.** Google Play **prohibits apps that download
> content from YouTube** (it violates YouTube's Terms of Service, and Play's Device and Network
> Abuse policy bans facilitating ToS violations of other services). It likewise prohibits apps
> that facilitate downloading of copyrighted content without authorization. A downloader scoped
> to YouTube or to "any site" with copyrighted catalogs is a **guaranteed rejection vector and a
> realistic account-level ban vector** — not a gamble worth taking on the operator's main
> developer account. Before ANY engineering or design spend, P1 must produce an explicit scope
> ruling: Play-safe scope, alt-store-only, or do-not-pursue. Every other section of this playbook
> is conditional on that ruling.

## Category Snapshot

- **Market reality.** Enormous raw demand — "video downloader" head terms are among the
  highest-volume utility queries on every store — but the Play-listed field is artificially
  thinned by policy enforcement. What survives on Play: downloaders scoped to platforms whose ToS
  permit it, "download your own content" tools, generic direct-URL/m3u8 fetchers, and in-app-browser
  file downloaders that carefully avoid platform-specific extraction. The aggressive end of the
  category lives off-Play: APK sites, Huawei AppGallery, RuStore, Aptoide — with looser but NOT
  absent copyright rules.
- **User intent.** "Save this video to my phone" — offline viewing, reposting, archiving own
  uploads. Intent is immediate and impatient; the user has a URL in the clipboard right now.
  Secondary intent: bulk/profile downloading (higher policy risk, higher willingness to pay).
- **Competition shape.** On Play: a rotating cast — apps get suspended and replaced continuously,
  so rankings churn and longevity itself is a moat for compliant apps. Off-Play: feature-race on
  supported sites, parallel downloads, and built-in players. User reviews concentrate on two
  failures: "site X stopped working" (extractor breakage) and ad overload.
- **Strategic posture for the factory:** treat this as a high-maintenance, high-risk,
  high-volume category. It is viable only with (a) a strictly compliant scope on Play, or (b) a
  deliberate alt-store strategy with the operator signing off on the risk.
- **Scope-vs-channel decision matrix** (the P1 output of this playbook):

| Scope | Google Play | AppGallery / RuStore | Direct APK | Maintenance burden |
|---|---|---|---|---|
| Own-content download (OAuth, official APIs) | Viable | Viable | Viable | Low — official APIs are stable |
| ToS-permitting platforms only | Viable with documented evidence | Viable | Viable | Medium |
| Generic direct-URL / m3u8 | Viable with neutral positioning | Viable | Viable | Low–medium |
| Per-platform extraction (popular social platforms) | **Not viable** | Risky; takedowns follow complaints | Possible | **High — weekly breakage** |
| YouTube in any form | **Banned — account-ban vector** | Prohibited on paper; complaint-driven takedowns | Legal exposure remains | High |

## Engineering Patterns

### Extraction layer (the fragile core)

- **Approaches, in ascending fragility:** (1) direct file URL / `<video src>` scraping — stable;
  (2) HLS/DASH manifest (m3u8/mpd) parsing + segment download + mux — stable tech, moderate
  per-site variance; (3) per-platform API/HTML extraction (the yt-dlp model) — **breaks weekly**
  as platforms change markup, signatures, and anti-bot measures. Any per-platform extractor is a
  permanent maintenance subscription, not a one-time build.
- **Architecture rule: extraction logic must be updatable without an app release.** Either a
  versioned rules/config file fetched from the developer's server, or a server-side resolver API
  (URL in → direct media URL out). Server-side resolution centralizes breakage fixes and hides
  logic, but puts the operator's server in the copyright liability chain — flag this in the P3
  risk register (`09-templates/risk-register.md`).
- **Library notes:** yt-dlp itself is Python (Unlicense) — running it on-device via Chaquopy is
  heavy and Chaquopy is commercially licensed; youtubedl-android wrappers exist but bundle the
  YouTube problem and large binaries. For Play-safe scopes prefer purpose-built Kotlin extraction:
  OkHttp (Apache 2.0) + jsoup (MIT) for HTML, Media3 (Apache 2.0) for manifest parsing.

### Download engine

- **WorkManager + foreground service, `foregroundServiceType="dataSync"`** (declare
  `FOREGROUND_SERVICE_DATA_SYNC` permission, targetSdk 34+ requires the typed declaration).
  Long downloads must survive process death; WorkManager gives retry/constraint handling,
  the foreground notification gives progress + cancel.
- **Resumable downloads:** HTTP `Range` requests with ETag/If-Range validation; persist
  bytes-downloaded per task in Room; resume after network loss (WorkManager `NetworkType`
  constraints + exponential backoff). For HLS: download segments individually (natural resume
  granularity), then mux to MP4 — Media3's transformer/muxer handles TS→MP4 without bundling
  ffmpeg. **Avoid ffmpeg-kit full builds:** GPL-configured variants are license traps and the
  upstream project's retirement makes maintained sources scarce; if ffmpeg is unavoidable, use an
  LGPL build, keep it dynamically isolated, and verify 16 KB page-size compliance of the `.so`.
- **MediaStore insertion:** write finished files via
  `MediaStore.Video.Media` with `IS_PENDING=1` during write, `RELATIVE_PATH` =
  `Movies/<AppName>`, clear `IS_PENDING` on completion — no storage permission needed for
  app-created media on API 29+. Never request `MANAGE_EXTERNAL_STORAGE` for this.

```kotlin
val values = ContentValues().apply {
    put(MediaStore.Video.Media.DISPLAY_NAME, fileName)
    put(MediaStore.Video.Media.MIME_TYPE, "video/mp4")
    put(MediaStore.Video.Media.RELATIVE_PATH, Environment.DIRECTORY_MOVIES + "/AppName")
    put(MediaStore.Video.Media.IS_PENDING, 1)
}
val uri = resolver.insert(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, values)!!
resolver.openOutputStream(uri)?.use { out -> /* stream segments */ }
resolver.update(uri, ContentValues().apply {
    put(MediaStore.Video.Media.IS_PENDING, 0)
}, null, null)
```

- Manifest requirements for the download service (targetSdk 34+):

```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<service android:name=".download.DownloadService"
         android:foregroundServiceType="dataSync" />
```

- P8 resilience tests, run verbatim:

```powershell
# Kill mid-download; download must resume, not restart at 0%
adb shell am kill com.example.app
# Doze simulation; queued downloads must complete after exit
adb shell dumpsys deviceidle force-idle
adb shell dumpsys deviceidle unforce
# Network flap during HLS segment fetch
adb shell svc wifi disable; Start-Sleep -Seconds 10; adb shell svc wifi enable
```

### Playback

- **Media3 ExoPlayer** (Apache 2.0) for the in-app player: local MP4s plus HLS preview before
  download. Trim the dependency to needed modules. A decent built-in player measurably lifts
  retention (the app becomes "where my videos live", not just a pipe).

## THE POLICY SECTION (decision gate — resolve in P1, revisit in P7)

**Prohibited on Google Play (do not build, do not imply in metadata):**

1. **Downloading from YouTube** in any form — including "via built-in browser". Violates YouTube
   ToS; Play enforces via Device and Network Abuse policy. Guaranteed rejection; repeat or
   flagrant cases escalate to **developer-account termination**, which kills every other app on
   the account.
2. **Facilitating unauthorized copyrighted downloads** — "download from any site" positioning,
   site lists full of copyrighted catalogs, or tutorial content showing copyrighted downloads in
   screenshots/video.
3. **Metadata that winks at it** — keywords/screenshots referencing prohibited platforms even if
   the app blocks them at runtime; reviewers judge the listing too.

**Play-viable scopes (each still needs verification against the named platform's CURRENT ToS
during P1 — ToS change; do not trust this list blindly):**

- Downloading the **user's own uploaded content** via official APIs/OAuth.
- Platforms whose ToS **explicitly permit** third-party downloading of public content (rare;
  verify per platform, in writing, dated, stored in `docs/factory/audits/DISCOVERY.md`).
- **Generic direct-URL / m3u8 download** with no platform-specific extraction and neutral
  positioning ("download files and streams you have the right to download") + a visible
  copyright/ToS disclaimer and a DMCA-style contact.
- Adjacent pivots with the same engine: podcast/lecture downloaders, personal-cloud media sync.

**Alt-store calculus.** Huawei AppGallery and RuStore review against their own content rules —
both still prohibit copyright infringement on paper, but enforcement on downloader apps is
materially looser; YouTube-scoped apps remain a takedown risk everywhere because rights-holder
complaints follow the app, not the store. See `04-aso/stores/huawei-appgallery.md` and
`04-aso/stores/rustore.md`. Direct-APK distribution removes store review entirely but also store
discovery — it suits an established brand, not a cold launch.

**Account hygiene.** If the operator pursues the aggressive end of this category at all, it goes
on a **separate developer account** with separate payment profile — Play bans are account-level
and association can spread them. This is an operator risk decision: present it explicitly in the
P1 output and record the decision in `docs/factory/PROJECT_STATE.md`; never default into it.

**Takedown-response runbook (have it written before launch, not after the email arrives).**
Suspensions, rights-holder/DMCA complaints, and trademark strikes are not "if" in this category;
they are "when". Pre-stage the response so panic doesn't drive a bad reply:

1. **Triage the notice type** within hours: Play policy suspension vs. DMCA/rights-holder
   complaint vs. trademark strike vs. AdMob serving-disable — each has a different appeal track
   and clock. Log it as a RISK-register event and in `docs/factory/PROJECT_STATE.md`.
2. **Do not silently re-publish** the same offending build/metadata — that escalates toward
   account termination. Fix the cited cause first (remove the platform reference, neutralize the
   browser default, scrub the screenshot), then appeal with the diff as evidence.
3. **For DMCA / rights-holder complaints:** if a server-side resolver is in the chain, the
   operator's infra is implicated (flagged earlier in this section) — have a counter-notice/contact
   path and a DMCA contact already published in the listing and app.
4. **Keep a dated evidence file** (ToS screenshots, scope ruling, neutral-positioning copy) from
   P1 onward — it is the spine of any appeal.

Detailed procedure, appeal-letter scaffolding, and the per-store escalation paths live in
`08-knowledge/stores/policy-landmines.md`; this is the category-specific trigger list that points
into it. P8 must re-read current Play policy text and confirm the runbook owner/contacts before
rollout.

## Design Patterns

- **Paste-URL hero flow.** Home screen = one large URL field + paste button + Download CTA.
  Auto-detect a media URL on the clipboard at app-open and offer it in one tap (with the
  Android 12+ clipboard-access toast in mind — read clipboard only on user-visible foreground,
  never silently in background). From paste to download-started must be ≤ 2 taps.
- **Quality/format picker** as a bottom sheet after resolution: resolutions, file sizes,
  audio-only option. Remember the last choice.
- **Queue & progress UX:** a Downloads tab listing active (progress %, speed, pause/cancel) and
  completed (thumbnail, play, share, delete) items, mirrored by the foreground-service
  notification. Failed items get an explicit retry with a human-readable reason.
- **In-app browser pattern:** common in the category (browse a site, sniff media, floating
  download button). Be aware of its **policy weight**: a built-in browser pointed at prohibited
  platforms is exactly what reviewers screenshot in rejection notices. If included on a Play
  build, restrict or neutral-default it, and never preload bookmarks to risky platforms.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Home / paste | The conversion moment | URL field + paste + clipboard auto-detect; ≤ 2 taps to download |
| Quality sheet | Control and trust | Resolutions, sizes, audio-only; remembers last choice |
| Downloads queue | Progress confidence | Speed, pause/cancel, retry with readable error |
| Player / library | Retention | Media3 player; downloaded items become the app's content |
| Settings | Power options | Location, Wi-Fi-only, parallel slots (pro), notifications |

- **Error UX is product UX here:** when extraction fails (it will), show "This link isn't
  supported yet" with a report action — never a spinner that hangs. Failure telemetry per
  domain feeds the maintenance loop.
- **Share-sheet entry:** register for `ACTION_SEND` of `text/plain` so users can share a URL
  straight into the app — this is the second-biggest entry point after paste and costs one
  intent filter.

## ASO Patterns

- **Keyword reality:** head terms ("video downloader", "video saver", "download videos") are
  high-volume and high-churn — suspended competitors constantly free up rank. Long-tail by
  capability: "m3u8 downloader", "hls downloader", "download video from link". **Never use
  platform trademarks** in title, short/long description, or keywords — trademark strikes are a
  separate enforcement track from content policy and arrive fast in this category. This also
  means no platform logos or recognizable platform UI in screenshots.
- **Title formula:** `Video Downloader — <safe scope>` (e.g. "Video Downloader — Save from Link")
  or `<Brand>: Video Saver & Player`. The scope phrase doubles as a compliance signal to
  reviewers.
- **Long description carries the compliance load:** state plainly what the app does and does not
  download, include the rights disclaimer, and avoid enumerating supported sites by brand name —
  describe capabilities ("direct links, HLS/m3u8 streams") instead. This paragraph is evidence
  in any review dispute.
- **Screenshot messaging:** 1 = paste-link flow with "Paste link → Download" caption; 2 = quality
  picker ("Choose quality, see size"); 3 = downloads queue with speed; 4 = built-in player;
  5 = Wi-Fi-only/background options. Use neutral demo content (own/CC-licensed video) only.
- Run `04-aso/workflows/competitor-analysis.md` with a policy lens: note which ranking
  competitors are non-compliant — their disappearance is your ranking opportunity, not your
  blueprint.
- **Metadata risk tiers** (apply during P7 keyword selection):

| Tier | Examples | Ruling |
|---|---|---|
| Safe | "video downloader", "download manager", "m3u8 downloader", "save video from link" | Use freely |
| Caution | "social video downloader", "story saver" | Only if the in-app scope genuinely and compliantly covers it |
| Forbidden | Any platform brand name or recognizable platform UI in any asset | Never — trademark strike + policy evidence |

- For alt-store listings, re-run keyword research per store — AppGallery and RuStore have their
  own search ecosystems and language splits; see `04-aso/stores/huawei-appgallery.md` and
  `04-aso/stores/rustore.md`.

- **Category benchmark (est.):** store-listing **CVR ~25–35%** on head terms — demand is intense
  and intent is immediate, so listings that survive policy convert well; alt-store CVR runs
  higher still where the category is less thinned. **Rating norm ~4.0–4.4** and structurally
  volatile: extractor breakage produces "stopped working" review waves that sink ratings between
  fixes, so a 4.0 here is healthier than a 4.0 in a stable category — read rating alongside the
  per-domain failure-rate telemetry, not in isolation. Record actuals against this in the
  ASO_HANDOVER Baseline Metrics Snapshot (`_V11_SPEC` D16).

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1 — all
  scope-neutral, no platform trademarks; the Stage-1 agent must NOT expand toward any platform
  brand):**

```text
video downloader, video saver, download videos, save video, video download,
download manager, file downloader, m3u8 downloader, hls downloader,
download video from link, save video from url, video downloader for android,
all video downloader, fast video downloader, hd video downloader, mp4 downloader,
download videos to phone, offline video, video grabber, link downloader,
stream downloader, media downloader, save videos offline, download video by url
```

## Monetization Patterns

- **Ad tolerance is unusually high** in this category — users accept ads as the price of a free
  downloader — but placement still decides ratings. What works: native/banner in the downloads
  list, interstitial **after a download completes** (the user got their value; cap 1 per N
  completions), rewarded ad to unlock a one-off premium action (e.g. 1080p, batch).
- **What churns: interstitial BEFORE the download starts.** Identical failure mode to the
  reader category's open-interstitial — the user has a task in hand; blocking it converts
  directly into uninstalls and 1-stars. Never gate the paste→download path with an ad.
- **Free/pro split:** free = standard-quality single downloads with ads. Pro (subscription works
  here given ongoing extractor maintenance; weekly/monthly tiers are common in the category) =
  no ads + max quality + **parallel/batch downloads + download speed (multi-connection)** —
  speed and parallelism are the proven pro levers. Paywall placement: at the quality picker
  (contextual upgrade) and on second concurrent download attempt.
- **Subscription price anchors (est. — validate per market in P7; the recurring revenue here is
  what funds permanent extractor maintenance):**

| Tier | Typical anchor (US, est.) | Notes |
|---|---|---|
| Weekly | **$2.99–$4.99/wk**, usually behind a 3-day free trial | The category's dominant tier. Annualizes to ~$150–$260/yr — deliberately steep; it monetizes the impatient one-off user who forgets to cancel. **Trial→paid auto-renew with weekly pricing is the single biggest refund/1★/chargeback driver and a Play subscription-transparency risk** — the trial terms, price, and renewal cadence must be unmissable on the paywall (Play requires this) or the listing eats "scam subscription" reviews. |
| Monthly | **$4.99–$7.99/mo** | The honest mid-tier; better LTV optics, fewer chargebacks than weekly. |
| Annual | **$19.99–$39.99/yr** | Best value framing; convert engaged users here. Show the per-week equivalent next to the weekly tier so the comparison is honest, not manipulative. |
| Lifetime / one-time | **$14.99–$29.99** | Viable where users distrust subscriptions; lower ceiling but converts the subscription-averse and reduces refund exposure. |

  Anchor the paywall on the annual tier with the weekly tier shown as the expensive comparator,
  never the reverse. Mediation/pricing detail: `08-knowledge/monetization/subscription-patterns.md`
  (this section overrides it where they conflict for this category).
- **Placement matrix:**

| Placement | Verdict | Rationale |
|---|---|---|
| Native/banner in downloads list | Allowed | Passive surface |
| Interstitial after download completes | Allowed, capped | Value already delivered; 1 per N completions |
| Interstitial before download starts | **Forbidden** | Task-blocking; the category's churn driver |
| Rewarded ad to unlock 1080p / one batch | Allowed | Honest exchange; also a paywall A/B lever |
| Ads during active download progress screen | Avoid | Anxiety surface; mis-taps cancel downloads |

- Mediation/ad-network details: `08-knowledge/monetization/ads-patterns.md`. Note ad networks
  also enforce content rules — AdMob can disable serving for copyright-facilitating apps
  independently of Play; an AdMob suspension on a risky build can bleed into the operator's
  AdMob account the same way Play bans do. Same separate-account logic applies.

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| YouTube or any-site download scope | Play policy | Rejection → suspension → account ban | P1 scope gate above; alt-store or do-not-pursue |
| Platform trademarks in metadata/screenshots | Trademark | Strike + listing takedown | Neutral naming, neutral demo content; P7 metadata audit |
| Extractor breakage (platforms change weekly) | Technical/ops | "Doesn't work" review floods within days | Remote-updatable extraction rules; monitoring + alerting on per-site failure rates; budget permanent maintenance |
| Account-level ban contaminating the operator's portfolio | Business | Catastrophic, irreversible | Separate dev account if pursuing risk scope — explicit operator decision logged in PROJECT_STATE |
| GPL ffmpeg build linked into closed-source app | License | Same trap as MuPDF for readers | LGPL build or Media3 muxer; license audit in P3 |
| Interstitial before download | Review-bomb | Churn + rating collapse | Post-completion placement only; verify in P8 |
| Silent clipboard reading in background | Privacy/policy | Clipboard-toast shaming, policy flag | Foreground-only clipboard checks |
| Downloads dying on process kill / Doze | Technical | "Stuck at 99%" reviews | WorkManager + typed foreground service + resumable Range logic; kill-test in P8 |
| In-app browser preloaded with risky platform bookmarks | Play policy | Screenshot evidence in rejection notices | Neutral default page; no risky bookmarks; consider removing browser on Play builds |
| Server-side resolver hosting copyrighted-extraction logic | Legal/business | Operator's server enters the liability chain | Log as a risk-register entry; scope resolver to compliant sources only |

## Recommended Workflow

1. **P1 Discovery — the decision gate (heaviest phase).** Determine actual extraction scope of
   the existing app, check every supported platform's ToS, classify: Play-safe / alt-store-only /
   do-not-pursue. Present the account-risk decision to the operator. Record ruling + evidence in
   `docs/factory/audits/DISCOVERY.md`. **Do not proceed to P3+ engineering spend without this
   ruling.**
2. **P3 Engineering Audit** — extractor inventory and breakage history; license audit (ffmpeg
   variants, wrappers); download-engine resilience (resume, process death); permission audit.
3. **P4 Modernization** — remote-updatable extraction architecture; WorkManager/dataSync engine;
   MediaStore insertion; Media3 player.
4. **P7 ASO (elevated, policy-led)** — metadata scrub for trademarks and prohibited-platform
   implications BEFORE keyword optimization; then keyword map and screenshots. For alt-store
   builds run the store-specific guides under `04-aso/stores/`.
5. **P5/P6 Design** — paste-hero flow and queue UX; standard weight.
6. **P8 Verification & Release** — policy self-review against the latest Play policy text (it
   shifts; re-read at release time, log the check in `docs/factory/reports/`), kill/resume tests,
   ad-placement check, staged rollout with review monitoring for "stopped working" spikes.

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`.
