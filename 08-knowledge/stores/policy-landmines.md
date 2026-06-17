# Policy Landmines — The Store Survival Guide

Evergreen catalog of the policy mistakes that get apps rejected, removed, or developer accounts
terminated — primarily Google Play, with AppGallery/RuStore deltas at the end. Agents consult
this during P1 (discovery — is the app category itself at risk?), P7 (ASO — are metadata claims
safe?), and P8 (release). For each landmine: the rule, how violations are detected, the penalty,
and the safe pattern. This file is doctrine; the executable pre-submission sweep is
`07-checklists/aso-launch.md` and `07-checklists/release-readiness.md`.

**Last reviewed: 2026-06-10.** Policy text, enforcement intensity, and permission regimes change
several times a year — **always verify against the current official Play policy pages and the
Play Console policy-status section before submission**. When this file and live policy disagree,
live policy wins.

---

## 0. How detection actually works (know your reviewer)

Play enforcement is layered; knowing which layer catches what tells you how much margin you have
(answer: none — but it dictates *when* violations surface):

1. **Submission-time automated scan**: manifest/permissions analysis, metadata text rules,
   binary static analysis, Data-safety-vs-traffic comparison. Catches §1 title rules and §2
   permission flags within hours. Deterministic — do not gamble against it.
2. **Submission-time human review**: triggered by sensitive permissions, sensitive categories,
   declarations, or random sampling. Reviewers install and use the app. Catches claim-vs-function
   gaps and minimum-functionality failures.
3. **Post-publication sweeps**: periodic category-wide re-reviews (recovery/cleaner/antivirus
   have been swept repeatedly) and re-scans triggered by policy updates. **An app approved last
   year is not safe this year** — policy currency is a standing maintenance duty.
4. **Complaint-driven review**: trademark holders, competitors, and users file reports that route
   to human review with elevated scrutiny. IP complaints are the fastest takedown path.
5. **Behavioral/statistical detection**: install-cohort anomalies, review-graph analysis,
   uninstall spikes (see `08-knowledge/aso/ranking-factors.md` §6).

Enforcement timeline expectation: automated rejections in hours; human-review rejections in
1–7 days; post-publication actions arrive with email lead time only for lower-severity issues —
severe violations (malware-class, IP, child safety) remove first, notify after.

## 1. Metadata & Claims

| Landmine | The rule | Detection | Penalty | Safe pattern |
|---|---|---|---|---|
| Misleading functionality claims | App must do what the listing says; "recover any deleted file", "boost speed 200%", antivirus/cleaner efficacy claims are under **active enforcement** in recovery/cleaner/security categories | Human review of claim vs behavior; user complaints; periodic category sweeps | Rejection; removal; metadata strike | Claim only what the app demonstrably does, with conditions stated ("recover photos that haven't been overwritten") |
| Trademark use in title/keywords | No third-party brand names in title, icon, or keyword-bait descriptions ("downloader for TikTok™" framing is contested space) | Trademark-holder complaints (DMCA-like), automated brand matching | Takedown; repeat = repeat-infringer track (§5) | Use generic category terms; brand compatibility statements only where nominative, never in title |
| "Best / Top / #1 / Free!" in title | Metadata policy bans promotional and performance superlatives in title; emojis and ALL-CAPS decoration too | Automated text checks at review | Rejection until fixed | Put differentiators as factual benefits in screenshots/description |
| Fake urgency / fear in listing | No alarmist claims ("your phone is infected", countdown pressure) in listing or screenshots | Human review; user reports | Rejection; deceptive-behavior strike if egregious | Problem→solution framing without manufactured threat (see `08-knowledge/aso/conversion-optimization.md` §4) |
| Unverifiable social proof | Invented user counts, awards, press quotes | Spot checks; competitor reports | Metadata strike | Only verifiable numbers (Play-visible installs, real ratings) |

## 2. Permissions

| Landmine | The rule | Detection | Penalty | Safe pattern |
|---|---|---|---|---|
| `QUERY_ALL_PACKAGES` | Allowed only for declared core use cases (launchers, device search, antivirus, banking…); requires a Console declaration + justification | Manifest scan at submission; declaration review | Rejection; removal of existing app at next update | Use `<queries>` element with specific packages/intents instead; almost no utility app qualifies for the blanket permission |
| `MANAGE_EXTERNAL_STORAGE` (All files access) | Declaration-or-rejection regime: permitted only when core functionality is file management/backup/antivirus-class and scoped APIs (SAF, MediaStore) demonstrably can't work | Automated manifest flag + human declaration review per update | Rejection of the release | File managers/recovery apps: declare with a precise core-functionality statement and an in-app fallback path; everyone else: SAF + MediaStore |
| SMS / Call Log permissions | Near-ban: only default-handler apps or narrow approved exceptions | Automated flag; exception review | Rejection; removal | Use SMS Retriever API for OTP; never request these in a utility app |
| Exact alarms (`USE_EXACT_ALARM` / `SCHEDULE_EXACT_ALARM`) | `USE_EXACT_ALARM` reserved for alarm-clock/calendar core functionality; `SCHEDULE_EXACT_ALARM` needs user grant and justification | Manifest scan + declaration | Rejection | `WorkManager`/inexact alarms for everything that isn't a user-facing alarm |
| Background location | Requires in-app prominent disclosure + Console declaration + video evidence of user benefit; reviewed strictly | Declaration review, behavioral analysis | Rejection; removal | Foreground-only location unless the feature is literally location-tracking-while-closed; ask incrementally (foreground first) |
| Photo/video broad access (`READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO`) | Broad media permissions allowed only when core functionality requires persistent/broad access; one-off picks must use the Android photo picker | Manifest scan + declaration review (regime active since 2024; verify current scope) | Rejection of update | Photo picker for selection flows; broad permission only for gallery/recovery-class core features, with declaration |
| `AD_ID` permission undeclared | Apps targeting modern API levels must declare the AD_ID permission if any SDK reads the advertising ID, and Data safety must match | Automated manifest + traffic check | Update block | Ads/analytics SDKs inject it — confirm via merged manifest and declare honestly |
| Full-screen intent misuse (`USE_FULL_SCREEN_INTENT`) | Reserved for genuinely time-critical alerts (alarms, calls); marketing notifications via FSI are banned | Declaration + behavioral review | Permission revoked at review; strike if abused | Standard notifications for everything else |
| Permission-purpose mismatch | Requesting permissions unused by actual functionality | Static + dynamic analysis at review | Rejection | Audit the merged manifest every release (`02-engineering/06-manifest-audit.md`); strip library-injected permissions with `tools:node="remove"` |

## 3. Content & Functionality

| Landmine | The rule | Detection | Penalty | Safe pattern |
|---|---|---|---|---|
| Downloader apps & copyrighted content | Apps must not facilitate downloading content in violation of source ToS — **YouTube is a hard wall** (its ToS forbids downloading; Play enforces); same logic increasingly applied to other platforms | Functionality review (reviewers test the app); rights-holder complaints | Removal; strike; repeated = account termination | Restrict to user-owned/public-domain/explicitly-permitted sources; block ToS-protected hosts at the app level; see `06-playbooks/video-downloader.md` for the compliant scope |
| Impersonation / clones | No copying another app's name, icon style, or implying affiliation; mass-produced template clones violate spam policy | Visual/text similarity matching; original-developer complaints | Removal; spam strikes hit the whole account | Distinct brand, distinct icon silhouette, substantive differentiation per app |
| Minimum functionality bar | Apps must be stable, installable, and do something useful; webview-wrappers and single-screen stubs fail | Human review; crash-at-review | Rejection | Ship the modernized, tested build; run `07-checklists/release-readiness.md` first |
| Deceptive ads / fake system UI | Ads must not imitate system dialogs, notifications, or app UI; no ads behind fake close buttons; full-screen ads must be closable | Automated creative scanning; user reports; ad-network policy sync | Ads policy strike; removal | Format rules in `08-knowledge/monetization/ads-patterns.md` §4 |
| Gambling-adjacent features | Real-money gambling needs licenses + restricted-country distribution; loot-box-like mechanics need odds disclosure; "casino-style" rewarded mechanics in non-gambling apps draw review | Category review, payment-flow analysis | Rejection; removal | Utility apps: avoid chance-based reward mechanics entirely; sweepstakes/raffle features need legal review first |

## 4. Data & Privacy

| Landmine | The rule | Detection | Penalty | Safe pattern |
|---|---|---|---|---|
| Data safety form inaccuracy | Declared collection/sharing must match actual app + SDK network traffic | **Automated traffic analysis** of the shipped binary vs the form; mismatch = enforcement | Warning → update block → removal | Generate the form from a real SDK inventory (`02-engineering/04-sdk-inventory.md`); re-verify every time an SDK is added/updated |
| Privacy policy URL | Required for every app; must be a live, accessible URL, specific to the app, covering all declared collection | Automated URL check + content review | Rejection | Host a real per-app policy; keep it versioned with SDK changes |
| Account deletion requirement | Apps with account creation must offer in-app account deletion AND a web deletion URL declared in Console | Form check + functional review | Update block | If the app doesn't truly need accounts, don't add them |
| SDK data-sharing liability | **You own your SDKs' behavior** — an ad/analytics SDK exfiltrating data = your violation; SDKs must meet Play's SDK requirements (self-update bans, etc.) | Binary analysis; Google's SDK index flags noncompliant versions | Strike against your app | Pin SDK versions consciously; check Google's SDK Index status before adopting; minimum SDK set per `08-knowledge/android/modern-stack.md` |
| Families / children flags | If the audience targeting includes children, the entire Families policy applies (ads SDKs must be certified, no behavioral ads…) | Audience declaration + content review | Removal | Utility apps: declare 18+/general adult audience honestly; never tick child categories for reach |

## 5. Account-Level (the existential tier)

- **Strike accumulation**: policy violations accrue per-app strikes; multiple strikes escalate to
  app removal and contribute to account-standing review. Strikes age out slowly — treat any
  strike as a Critical incident, document root cause in the target's `docs/factory/CHANGELOG.md`,
  and file the appeal within the stated window.
- **Associated-account bans (the contagion rule)**: if an account is terminated, **any account
  Google links to it — by payment method, device, IP pattern, shared APK signatures, listed
  developers — is subject to termination too**. A banned partner, a purchased app from a banned
  developer, or shared infrastructure can kill an otherwise clean account. Never buy/transfer
  apps without provenance checks; never share credentials/payment instruments across accounts.
- **Repeat-infringer policy**: multiple IP takedowns across any apps in the account trigger
  account-level termination, mirroring DMCA repeat-infringer doctrine. Trademark/copyright
  complaints are therefore account-level threats, not app-level annoyances.
- **Account verification & dormancy**: developer identity verification (D-U-N-S for orgs),
  20-tester/14-day requirements for new personal accounts, and dormant-account closure policies
  apply — verify current onboarding requirements; they have tightened repeatedly.
- **Appeals**: one well-evidenced appeal beats three emotional ones. State the violation, the
  exact remediation shipped (versionCode), and the prevention measure. Keep all correspondence.

## 6. Per-store deltas

- **Huawei AppGallery** (`04-aso/stores/huawei-appgallery.md`): notably **strict on copyright
  documentation** — expect to upload proof of rights for content, fonts, characters, and brand
  elements, and software copyright certificates for some regions (mainland China requires local
  publisher + ICP-class paperwork). Review is human-heavy and slower; functionality must match
  the declared description closely. Privacy compliance review mirrors Play's but with manual
  document checks.
- **RuStore** (`04-aso/stores/rustore.md`): content rules follow Russian law — age-rating
  labeling per local classification, restrictions on content categories regulated by RF law
  (verify the current list with RuStore moderation docs), and Russian-language listing
  effectively required. Moderation actively compares metadata claims against real functionality.
  Sanctions-related constraints affect payment rails, not content review, but shape what
  monetization you can declare (see `08-knowledge/stores/store-comparison.md`).
- Common to both: smaller review teams mean **rejections come with usable human feedback** —
  read it literally and resubmit precisely; arguing categories is futile.

## 7. Severity triage for audit findings

When a P1/P3/P7 audit surfaces a policy gap in a target app, grade it on the factory severity
scale (`01-core/PROJECT_LIFECYCLE.md` conventions) as follows:

| Finding class | Severity | Rationale |
|---|---|---|
| Anything in §5 territory (IP complaints pending, associated-account exposure, accumulated strikes) | Critical | Account-existential |
| Live violation in a swept category (recovery-claim overreach, downloader scope, restricted permission shipping without declaration) | Critical | Removal-class on next sweep |
| Data safety form mismatch with actual SDK traffic | High | Enforcement is automated and certain, but graduated (warning first) |
| Metadata superlatives, missing privacy URL, unverifiable social proof | High | Blocks next submission |
| Stale declarations (permission declared but flow changed), aging consent implementation | Medium | Latent — fails at next human review |
| Listing/claims drift from current feature set | Medium | Conversion + claim risk compounding |

Record all of these in the target's risk register (`09-templates/risk-register.md`) with an
owner and a re-verify date — policy findings decay; a "safe" determination is only valid until
the next policy update cycle (Google announces changes with effective dates several times a
year; subscribe to the Play Console policy-update feed).

## 8. Operating doctrine

1. **Audit category risk at P1.** Recovery, cleaner, antivirus, downloader, and VPN categories
   carry standing enforcement attention — plan claims and permissions defensively from day one
   (playbooks in `06-playbooks/` encode per-category specifics).
2. **The merged manifest is the contract.** Audit it every release (`02-engineering/06-manifest-audit.md`);
   libraries inject permissions you'll be punished for.
3. **The Data safety form is generated, not remembered.** Source it from the SDK inventory
   artifact, never from memory of "what we collect".
4. **Run the pre-submission sweep** in `07-checklists/aso-launch.md` (metadata/claims layer) and
   `07-checklists/release-readiness.md` (technical/policy layer) before every store submission —
   no exceptions for "small" updates; automated review re-scans everything each time.
5. **Treat policy mail as production incidents**: respond within 7 days, fix before deadline,
   log in `docs/factory/CHANGELOG.md`, update the risk register (`09-templates/risk-register.md`).
