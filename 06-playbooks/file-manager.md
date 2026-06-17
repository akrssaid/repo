# File Manager Playbook

Category playbook for file manager / file explorer apps (general-purpose managers, dual-pane
explorers, storage analyzers/cleaner-adjacent tools, network-share browsers). Load during P1 per
`06-playbooks/README.md` and keep loaded through release. The category's two existential facts:
(1) it is one of the few categories where Play **permits** `MANAGE_EXTERNAL_STORAGE` — but only
through a declaration-form approval that must be earned, and (2) its reputation is binary — a
file manager that ever loses user data does not recover its rating.

## Category Snapshot

- **Market reality.** Mature, high-volume utility category. Files by Google is the preinstalled
  default; the third-party tier is led by Solid Explorer, MiXplorer, Total Commander, Cx File
  Explorer, and FV/ZArchiver for archives. The ES File Explorer ban (adware/click-fraud) reset
  the category's trust baseline years ago — "clean, no shady ads" is now an explicit purchase
  criterion in reviews.
- **User intent.** Task-driven, in clusters: (a) find/open a specific file ("where did that
  download go"), (b) move/copy — especially to SD card or USB OTG, (c) free up space (the
  storage-analyzer intent — the biggest growth/retention intent), (d) extract archives,
  (e) access network shares (SMB to a PC/NAS). Power users add root/dual-pane intents but are a
  small, vocal minority.
- **Competition shape.** Files by Google owns the casual tier; third parties win on power
  features (network, archives, dual-pane, customization) and on doing scoped storage *well*.
  The decisive review themes: data-loss horror stories (negative) and "does everything, no ads
  in the way" (positive).
- **Retention reality.** The analyzer/cleaner loop ("what's eating my storage") is the recurring
  re-entry use case; pure on-demand file ops alone produce shallow retention.
- **Competitive reference set** (study during P1 and `04-aso/workflows/competitor-analysis.md`):

| App | What it proves | What to copy / avoid |
|---|---|---|
| Files by Google | Casual tier is owned by the default | Compete on power features it refuses to add |
| Solid Explorer | Paid/pro model works for network + dual-pane | Copy: pro split, material polish |
| MiXplorer | Feature depth builds cult loyalty off-Play | Note: its audience expects density, not minimalism |
| Total Commander | Dual-pane veterans are a real, paying niche | Copy: keyboard-grade efficiency in selection ops |
| Cx File Explorer | Clean free app with network shares ranks well | Copy: approachable network setup UX |
| ES File Explorer (banned) | Ad SDK greed kills 500M-install apps | The permanent cautionary tale |

## Engineering Patterns

### Storage access matrix (the architecture-defining decision)

| API level | Reality | What to use |
|---|---|---|
| 26–28 | Legacy external storage; `READ/WRITE_EXTERNAL_STORAGE` give broad `File` API access | Direct `java.io.File` ops |
| 29 | Scoped storage introduced; `requestLegacyExternalStorage` opt-out works | Opt out for 29 only; plan forward |
| 30+ | Scoped storage enforced; broad `File` access requires `MANAGE_EXTERNAL_STORAGE` (all-files access) | All-files access (if approved) or SAF trees |
| 30+ `Android/data`, `Android/obb` | Hidden from all-files access AND from SAF tree picking (33+ blocks granting these trees entirely) | No reliable access; do not promise it |
| 33+ | Granular `READ_MEDIA_IMAGES/VIDEO/AUDIO` replace `READ_EXTERNAL_STORAGE` for media-only access | Only relevant if shipping a media-browser mode without all-files |

**`MANAGE_EXTERNAL_STORAGE` on Play — permitted for genuine file managers, by procedure:**

1. Core functionality must BE file management (Play's named allowed use cases include file
   managers, backup, and antivirus). The app must be unusable without broad access — and the
   listing/screenshots must make that obvious to a reviewer.
2. Declare the permission in the manifest and complete the **All files access declaration** in
   Play Console (App content → Sensitive app permissions): select the core use case, justify in
   one or two factual sentences, and supply/refresh a short demo video showing the feature
   needing it.
3. In-app: check `Environment.isExternalStorageManager()`; request via
   `ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION` with a prior in-app explanation screen (the system
   page gives no context). Handle denial gracefully — degrade to SAF, don't dead-end.
4. Every app UPDATE re-passes review with the declaration; a pivoted or bloated feature set can
   get a previously approved app rejected. Keep the core-function story stable.

```kotlin
// Gate, explain, then deep-link to the system toggle
if (!Environment.isExternalStorageManager()) {
    // show in-app rationale screen first, then:
    startActivity(Intent(
        Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION,
        Uri.parse("package:$packageName")))
}
```

```powershell
# Emulator-only shortcut for development; production must use the system flow
adb shell appops set --uid com.example.app MANAGE_EXTERNAL_STORAGE allow
```

5. Use **SAF for everything peripheral**: SD-card write on weird OEMs, USB OTG, user-chosen
   backup destinations. SAF trees (`ACTION_OPEN_DOCUMENT_TREE` + persisted permissions) are also
   the complete fallback architecture if the declaration is ever rejected.

### MediaStore vs File API decision table

| Need | Use |
|---|---|
| Browse arbitrary directories, hidden files, any extension | `File` API (requires all-files access on 30+) |
| Media-gallery views, "Images/Videos/Audio" category cards | MediaStore queries (fast, indexed, no all-files needed) |
| Sizes/dates for analyzer over media | MediaStore first (cheap), `File` walk for non-indexed remainder |
| Writing into standard collections without broad permission | MediaStore insert with `RELATIVE_PATH` + `IS_PENDING` |
| SD/OTG/user-chosen destinations | SAF `DocumentFile` (slower — batch ops should minimize per-file round trips) |

After bulk `File`-level changes, trigger `MediaScannerConnection.scanFile` so galleries and other
apps see reality.

### Core operations engine

- **Copy/move/delete as WorkManager jobs with a typed foreground service** (`dataSync`) and a
  progress notification (current file, x of y, bytes/sec, cancel). Operations must survive
  process death and screen-off.
- **Transactional discipline (non-negotiable):** the move algorithm, exactly:
  1. Enumerate sources; pre-flight free-space and permission checks on the destination.
  2. Copy each file to the destination under a temp name (`.tmp-<uuid>`).
  3. Verify: size match always; optional checksum mode for user-critical batches.
  4. Atomically rename temp → final name (resolving conflicts per the queued policy).
  5. Only after the entire batch verifies: delete sources (same-volume `renameTo` fast-path is
     the one exception — it is already atomic).
  6. On cancel/crash at any step: sources untouched; orphaned `.tmp-*` files cleaned on next
     launch.
  Never delete-as-you-go. Cross-filesystem moves (internal→SD via SAF) are always
  copy+verify+delete.
- **Conflict resolution:** on name collision present Skip / Replace / Keep both (auto-rename
  `name (1).ext`) / and "apply to all" — queued per batch, asked once up front where possible,
  not 400 modal dialogs mid-operation.
- **Archives:** ZIP via built-in `java.util.zip` (or Zip4j, Apache 2.0, for AES-encrypted zips).
  7z: Apache Commons Compress (Apache 2.0, pure-Java, slower) or XZ/LZMA libs. **RAR is the
  license trap:** `junrar` (extract-only) carries the unrar license restriction (must not be used
  to recreate the RAR compressor — acceptable for extraction, note it in the license inventory);
  full librar/WinRAR code is non-redistributable. Never promise RAR *creation*.
- **Network shares:** SMB via SMBJ (Apache 2.0; SMB2/3 — avoid jcifs-legacy/SMB1) or jcifs-ng
  (LGPL — dynamic-link care for closed source); FTP/FTPS via Apache Commons Net (Apache 2.0);
  SFTP via sshj (Apache 2.0; JSch original is BSD-style but stale — use the maintained
  `com.github.mwiede:jsch` fork); WebDAV via sardine-android (Apache 2.0). Stream through a
  unified VFS abstraction so the ops engine is source-agnostic.
- **Root features: avoid on Play builds.** Root explorers trip Play's malware/elevated-privilege
  scrutiny, break on modern devices, and serve a sliver of users. If ever shipped, make it an
  off-Play variant.

### Performance traps

- Directory listing on huge folders (10k+ entries): page the listing, lazy-load icons/thumbnails
  (Coil with custom fetchers), never `listFiles()` on the main thread.
- Storage-analyzer full-tree walk is expensive: run in a coroutine worker, cache results with
  timestamps, incremental refresh; don't re-walk on every screen visit.
- SAF `DocumentFile` per-call IPC overhead: for batch ops use `DocumentsContract` directly with
  bulk queries instead of `DocumentFile` convenience calls.

## Design Patterns

- **Home: category cards + storage gauge,** not a bare directory listing. Top apps converge on:
  storage-used bar/ring up top, category cards (Images, Videos, Audio, Documents, Downloads,
  Apps, Archives), then Internal storage / SD / network shortcuts, then Recent files. This serves
  the casual majority; the path tree is one tap deeper.
- **Single list default, dual-pane as a power option** (tablet/landscape default, opt-in on
  phones). Dual-pane is a differentiator for the Total Commander audience — gate it behind
  settings or pro, don't force it on everyone.
- **Breadcrumbs** for the current path, horizontally scrollable, each segment tappable; long-press
  to copy path. Back button walks up the tree before exiting.
- **Selection mode conventions:** long-press enters selection; tap toggles; select-all and range
  select in the action bar; contextual actions (copy/move/delete/share/rename/compress) in a
  bottom action bar. Counts in the title.
- **Storage analyzer is THE retention feature:** category-size breakdown chart, **large-files
  finder** (sorted descending), duplicate finder, "unused/old downloads" suggestions. Make it a
  first-class tab/destination, not a buried tool. Always show what will be freed and require
  explicit confirmation before any deletion.
- **Search:** global with scope toggle (current folder vs everywhere), filters by type/size/date;
  indexed (Room FTS over a maintained file index) if all-files access is granted, else
  MediaStore-backed. Show results streaming in as found — a spinner followed by a complete list
  feels slower than incremental results, and search latency is a named complaint in category
  reviews.
- **Accessibility floor:** selection mode fully operable with TalkBack (announce
  selected counts), 48 dp targets on list action icons, breadcrumb segments focusable — run
  `03-design/09-accessibility-review.md` against the browser list specifically.
- **Key screens and their jobs:**

| Screen | Job | Notes |
|---|---|---|
| Home dashboard | Orientation + analyzer entry | Storage gauge, category cards, sources, recents |
| Browser list | The work surface | Sort/view toggles, breadcrumbs, selection mode |
| Analyzer | Retention | Size chart, large-files finder, duplicates, confirm-before-delete |
| Transfer progress | Trust at the riskiest moment | Per-file progress, speed, cancel that leaves source intact |
| Network sources | Power differentiator | Add SMB/SFTP/FTP/WebDAV; saved credentials in EncryptedSharedPreferences/Keystore |
| Search | Findability | Scope toggle, type/size/date filters |
| Settings | Control | View modes, theme, hidden files, root of safety prompts |

- **Empty and error states carry trust:** an unreadable SD card or a dropped SMB share must say
  exactly what happened and what to try — silent empty lists read as data loss to users.

## ASO Patterns

- **Keyword themes by intent tier:**

| Tier | Examples | Competition | Role |
|---|---|---|---|
| Head | "file manager", "file explorer", "files" | Brutal (Files by Google + incumbents) | Title presence; long game |
| Task phrasing | "move files to sd card", "extract zip file", "usb otg file manager" | Medium | Long-description natural sentences; realistic early rankings |
| Power/network | "ftp client", "smb file share", "webdav android" | Low–medium | Differentiator coverage; converts power users |
| Analyzer adjacency | "storage analyzer", "free up space", "large files finder" | Medium, policy-watched | Use mechanical phrasings only (see caution below) |
- **Cleaner-claims caution:** storage-cleaner keywords are policy-watched territory. Claims must
  be mechanical and truthful — "find and delete large files" is fine; **"boost your phone",
  "speed up", "RAM cleaner", battery-magic pseudo-claims are misleading-claims bait** and are
  associated with the disruptive-ads/cleaner crackdown cohort. Keep all such phrasing out of
  title, description, and screenshots.
- **Title formula:** `<Brand> File Manager: Explorer & Cleaner-safe-term` — practical patterns:
  "File Manager — Files & SMB", "<Brand>: File Explorer & Storage Analyzer". Lead with "File
  Manager" or "File Explorer" (split the pair across title and short description to cover both).
- **Listing-as-declaration-evidence:** the same reviewer pool that judges the all-files
  declaration sees the listing. Screenshots and description must make "this is a file manager"
  unmissable — burying file management under cleaner/analyzer marketing weakens the
  permission case. Write the declaration justification and the listing in the same P7 pass.
- **Screenshot messaging:** 1 = home with storage gauge ("All your files, organized"); 2 =
  analyzer/large-files ("See what's eating your space"); 3 = transfer progress with conflict
  dialog ("Move safely — nothing lost"); 4 = network shares ("Your NAS and PC, in your pocket");
  5 = archives; 6 = dark theme/dual-pane. Trust language ("no ads in your files", if true)
  converts in this post-ES-Explorer market.

- **Review-reply angle (a trust lever in a trust-scarred category).** Post-ES-Explorer, listing
  visitors read developer replies as evidence of who's running the app. Two reply classes earn
  the most: (1) **data-loss panic** ("my files disappeared after a move!") — reply fast, calm,
  and concretely (most "disappeared" reports are unindexed files needing a MediaScanner pass, or
  a move that completed to SD/network; walk them to the file and confirm the transactional engine
  never deletes before verify — this directly counters the category's worst fear and often flips
  the rating); (2) **ad/permission complaints** — affirm what the all-files permission is for and
  that there are no ads in the file views (if true). Triage cadence, escalation, and tone rules
  live in `04-aso/workflows/ratings-reviews.md`; template the data-loss reassurance and the
  all-files-rationale strings into `docs/factory/assets/` during P7 and wire them to the
  ASO_HANDOVER Review-Response Readiness section (`_V11_SPEC` D16).

- **RU/AG keyword-seed note — network/SMB terms diverge by market.** AppGallery (Huawei) and
  RuStore audiences are disproportionately power/NAS users, and their search vocabulary differs
  from the en seed list: re-run `04-aso/workflows/keyword-research.md` per store rather than
  translating en seeds. Russian seeds users actually type: "файловый менеджер", "проводник",
  "доступ к SMB", "подключить NAS", "FTP клиент", "архиватор", "распаковать архив", plus the
  Latin "SMB"/"NAS"/"FTP"/"WebDAV" acronyms (kept Latin — do not Cyrillicize). AppGallery skews
  toward Chinese-origin NAS ecosystems, so "NAS", "Samba/SMB", and brand-neutral "网络存储"
  (network storage) style intents over-index — keep "smb"/"nas"/"ftp"/"webdav" as Latin
  long-tail seeds in every locale because power users search the protocol name directly. See
  `04-aso/stores/huawei-appgallery.md` and `04-aso/stores/rustore.md`.

- **Category benchmark (est.):** store-listing **CVR ~18–28%** — head-term traffic is broad and
  partly casual (many convert on the preinstalled Files by Google instead), so third-party
  listings convert toward the lower band unless a differentiator (network/dual-pane/analyzer) is
  front-loaded in the screenshots. **Rating norm ~4.3–4.6**; the trust-positioned paid apps
  (Solid Explorer) sit highest, and any data-loss review pattern pulls a listing below the floor
  fast. Record actuals against this in the ASO_HANDOVER Baseline Metrics Snapshot (`_V11_SPEC` D16).

- **Keyword seed list (feed verbatim into `04-aso/workflows/keyword-research.md` Stage 1 — keep
  the protocol acronyms as forced long-tail inclusions; scrub all cleaner pseudo-claims per the
  caution above):**

```text
file manager, file explorer, files, file browser, files manager,
move files to sd card, copy files, extract zip file, unzip, zip file opener,
usb otg file manager, storage analyzer, free up space, large files finder,
duplicate file finder, ftp client, smb file share, nas browser, webdav,
sftp client, archive manager, rar extractor, 7z opener, hidden files,
sd card manager, document manager, file transfer, dual pane file manager
```

## Monetization Patterns

- **What works:** native ads **in file lists, clearly labeled "Ad", visually distinct from file
  rows, and never adjacent to action buttons or positioned where a list item would be at the
  moment of a tap** (accidental-click layouts are both a Play ads-policy violation and a rating
  killer). Banner on the home dashboard is acceptable; interstitials at most on
  analyzer-completion (a natural pause), frequency-capped.
- **Never:** ads inside transfer-progress or delete-confirmation flows; full-screen ads
  interrupting an operation; notification ads (banned outright).
- **Free/pro split:** free = full local file management with ads (crippling core file ops behind
  a paywall reads as hostile in this category and reviews punish it). Pro (one-time purchase is
  the category norm; subscriptions meet resistance) = no ads + **themes/icon packs + network
  sources (SMB/SFTP/WebDAV) + dual-pane + analyzer extras (duplicates)**. Paywall placement:
  contextual, on tapping a pro feature (e.g. adding a network source).
- **Placement matrix:**

| Placement | Verdict | Rationale |
|---|---|---|
| Native ad in file list (labeled, distinct, away from actions) | Allowed | Passive browsing surface |
| Banner on home dashboard | Allowed | Lowest-friction slot |
| Interstitial on analyzer completion | Allowed, capped | Natural pause after value delivery |
| Ad in transfer-progress or delete-confirm flow | **Forbidden** | Mis-tap = cancelled transfer or accidental delete |
| Notification ads | **Forbidden** | Banned by Play outright |
| App-open ad | Caution, cap 1/session | File managers are opened mid-task; heavy cost to perceived speed |

- Cross-reference `08-knowledge/monetization/ads-patterns.md`; the ES File Explorer history is
  the category's cautionary tale — aggressive ad SDKs killed a 500M-install app.

## Common Pitfalls

| Pitfall | Type | Consequence | Mitigation |
|---|---|---|---|
| All-files-access declaration rejected (weak justification, file management not obviously core) | Play policy | Update blocked until resolved | Follow the declaration procedure above; keep demo video current; maintain SAF fallback architecture |
| Data-loss bug in move/copy (delete-before-verify, crash mid-move) | Technical/reputation | 1-star avalanche; category reviews never forgive it | Transactional ops engine; copy→verify→delete; P8 kill-mid-transfer test matrix |
| `Android/data`/`Android/obb` promises on Android 11+ | Technical/honesty | "Broken" reviews; no workaround is Play-viable | Don't list it as a feature; show an honest explainer in-app when users navigate there |
| "Boost/speed up/battery" cleaner pseudo-claims | Play policy (misleading claims) | Listing rejection; cleaner-crackdown cohort flagging | Mechanical, truthful claims only; P7 metadata audit |
| Antivirus claims without AV capability | Play policy | Misleading-functionality flag | Don't bolt "virus cleaner" keywords onto a file manager |
| Ad placed where a file row belongs / near Delete | Ads policy + UX | Accidental clicks → policy strike; rage reviews | Labeled, visually distinct native slots away from action zones |
| RAR-creation promise or bundled non-redistributable rar code | License | Legal exposure | Extract-only via junrar with license note; never offer RAR compression |
| SMB1 (jcifs-legacy) network code | Technical/security | Fails on modern NAS/Windows; security flags | SMBJ/jcifs-ng (SMB2/3) only |
| Full-tree rescans on every open | Performance | ANRs, battery complaints | Cached, incremental analyzer indexing |
| Network credentials stored in plaintext prefs | Security | Data-safety form violation; audit findings | Android Keystore / EncryptedSharedPreferences; declare in Data safety form |
| Forgetting MediaScanner after bulk File-API changes | Technical | "My photos disappeared" panic reviews (they're just unindexed) | `MediaScannerConnection.scanFile` after every batch op |
| Analyzer auto-deleting without explicit confirm | UX/trust | Indistinguishable from a data-loss bug to the user | Every deletion behind a summary + confirm; offer app-side recycle bin |

## Recommended Workflow

1. **P1 Discovery** — establish the storage-access posture of the existing app (legacy storage?
   all-files declared and approved? SAF-only?) and whether the analyzer loop exists. The access
   posture decides the modernization roadmap's spine.
2. **P3 Engineering Audit (heavy)** — storage matrix compliance per API level; ops-engine
   transactionality audit (find every delete-before-verify path — each is a Critical finding);
   archive/network library license inventory; ad-SDK behavior audit (the ES lesson).
3. **P4 Modernization** — scoped-storage architecture (all-files + SAF fallback), WorkManager ops
   engine with transactional semantics, MediaStore-backed category views.
4. **P5/P6 Design & Redesign** — dashboard home + analyzer as first-class feature; selection-mode
   and conflict-dialog polish; these drive both rating and retention.
5. **P7 ASO** — task-phrase keyword map; scrub all cleaner pseudo-claims; trust-forward
   screenshots; prepare the all-files declaration text alongside the listing (same reviewer
   audience).
6. **P8 Verification & Release** — kill-mid-transfer test matrix (process death, storage
   ejection, network share drop during copy), all-files declaration dry-run check, ad-placement
   distance-from-action audit, staged rollout watching for data-loss reports as release-blockers.

Category-specific P8 verification additions (append to
`07-checklists/release-readiness.md` items):

```text
[ ] Move of 1 GB / 5,000 files internal→SD completes; source intact until verify passes
[ ] adb shell am kill mid-move: source files untouched, partial targets cleaned on relaunch
[ ] SD card ejected mid-copy: graceful error, zero source loss
[ ] Conflict dialog: Skip / Replace / Keep both / Apply-to-all all function in a 100-collision batch
[ ] 10,000-entry folder lists without ANR (StrictMode + adb shell dumpsys gfxinfo check)
[ ] All-files declaration text, demo video, and listing screenshots tell the same core-function story
[ ] No ad renders within one row height of Delete/Move in any list state
```

Phase definitions: `01-core/PROJECT_LIFECYCLE.md`. Policy cross-reference:
`08-knowledge/stores/policy-landmines.md`.
