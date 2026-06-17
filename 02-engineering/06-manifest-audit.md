# Manifest Audit Method

This method audits the **merged** release manifest for permission overreach, unsafe exported
components, intent-filter hygiene, network-security posture, backup-rule exposure, and task/launch
smells. It runs in P3 under `01-core/prompts/03-modernization-audit.md` and is re-run at P8 by
`01-core/prompts/15-final-verification.md` (the manifest that ships is the merged one — libraries
inject permissions and components you never wrote). Every finding here carries direct Play-policy
or security weight; severities below are calibrated accordingly.

## Step 0 — Obtain the Merged Manifest (canonical statement)

**This step is the single source of truth for the merged-manifest path. All factory docs reference
this step instead of restating the path.**

```bash
./gradlew :app:processReleaseManifest    # substitute the variant: process<Variant>Manifest
```

The output path depends on the AGP version (read it per `02-engineering/01-discovery.md` Step 3):

| AGP version | Merged-manifest path |
|---|---|
| ≤ 7.2 (older AGP) | `app/build/intermediates/merged_manifests/<variant>/AndroidManifest.xml` |
| 7.3+ (newer AGP, incl. all 8.x) | `app/build/intermediates/merged_manifests/<variant>/process<Variant>Manifest/AndroidManifest.xml` |

`<variant>` is lowercase (`release`, `debug`, `gplayRelease`); `<Variant>` in the task-named
subdirectory is capitalized (`processGplayReleaseManifest`). If neither path exists, search:
bash `find app/build/intermediates/merged_manifests -name AndroidManifest.xml`; powershell
`Get-ChildItem -Recurse app\build\intermediates\merged_manifests -Filter AndroidManifest.xml`.

Alternatives when Gradle is unavailable or you need the *packaged* truth:

```bash
# From a built artifact (binary manifest, decoded):
aapt2 dump xmltree app/build/outputs/apk/release/app-release.apk --file AndroidManifest.xml
# Or in Android Studio: Build > Analyze APK… > AndroidManifest.xml
# Merge provenance (which library injected what):
# app/build/outputs/logs/manifest-merger-release-report.txt
```

Always audit the merged file; diff it against `src/main/AndroidManifest.xml` and attribute every
addition to its source library via the merger report. Unattributable additions are themselves a
High finding. In the commands below, `$MERGED_MANIFEST` = the path resolved in this step.

## Step 1 — Permission Audit

List every `<uses-permission>`. For each, fill:

| Permission | Class | Requested by (app/library) | Used where (file:line) | Justified? | Action |
|---|---|---|---|---|---|

Classes: **normal** (install-time, low scrutiny), **dangerous** (runtime grant), **special**
(settings-screen grant), **signature**. Find usage:

```bash
grep -rn --include="*.kt" --include="*.java" "checkSelfPermission\|requestPermissions\|RequestPermission(" app/src
```

```powershell
Get-ChildItem -Recurse app\src -Include *.kt,*.java |
  Select-String "checkSelfPermission|requestPermissions|RequestPermission\("
```

**High-scrutiny table** — these need explicit justification or removal; several require Play
Console declaration forms and can get the app rejected:

| Permission | Policy weight | Keep only if |
|---|---|---|
| `QUERY_ALL_PACKAGES` | Play-restricted; declaration form; common rejection | Core function requires enumerating all apps (launcher, antivirus). Otherwise use `<queries>` element with specific packages/intents |
| `MANAGE_EXTERNAL_STORAGE` | Play-restricted; declaration form | File manager / backup core function. Otherwise SAF or MediaStore (`08-knowledge/stores/policy-landmines.md`) |
| `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` | Declaration form; `USE_EXACT_ALARM` only for alarm/calendar core function | Exact-time user-facing events; see `02-engineering/05-service-inventory.md` |
| `ACCESS_BACKGROUND_LOCATION` | Declaration + video review | Background location is the product (tracking app) |
| `READ_MEDIA_IMAGES/VIDEO` vs `READ_EXTERNAL_STORAGE` | Granular media required on 33+ | App reads user media; prefer Photo Picker (no permission) where possible |
| `READ_PHONE_STATE`, `READ_CONTACTS`, `READ_SMS`, `RECORD_AUDIO`, `CAMERA` | Dangerous; SMS/Call-log are Play-restricted families | Used by a real feature with runtime flow; SMS/Call-log almost never survive review |
| `REQUEST_INSTALL_PACKAGES` | Restricted | App is a store/installer (RuStore-distributed updaters sometimes need it — document per store) |
| `AD_ID` | Data-safety linkage | Any ads/analytics SDK keeps it; if app is ad-free, exclude: `<uses-permission android:name="com.google.android.gms.permission.AD_ID" tools:node="remove"/>` |

Also flag: `maxSdkVersion` opportunities (e.g., `WRITE_EXTERNAL_STORAGE` with
`android:maxSdkVersion="28"`), and permissions present with **zero** usage hits (Library-injected
overreach — remove with `tools:node="remove"`).

## Step 2 — Exported Components Audit

Since API 31, every component with an `<intent-filter>` **must** declare `android:exported`
explicitly (build fails otherwise — legacy projects pre-targetSdk-31 may still lack it). Audit
every component (inventory tables come from `02-engineering/05-service-inventory.md`):

| Check | Rule | Severity if violated |
|---|---|---|
| Explicitness | Every activity/service/receiver with intent filters has explicit `exported` | High (blocks targetSdk raise) |
| Launcher only | Exactly the launcher activity (and legitimate share/deep-link targets) are `exported="true"` | — |
| Exported without permission guard | Any exported non-launcher component lacking `android:permission` — can it be invoked maliciously? Read its `onCreate`/`onReceive` for unvalidated extras | High; Critical if it performs privileged actions (file write, payments, auth) |
| Exported providers | `<provider android:exported="true">` without `readPermission`/`writePermission` | Critical |
| FileProvider | Must be `exported="false"` with `grantUriPermissions="true"`; check `file_paths.xml` for over-broad `<root-path>` | High if root-path |
| Library-injected exported components | Attribute via merger report; verify vendor guidance | Medium |

## Step 3 — Intent Filter Hygiene

- Deep-link filters: every `<data>` scheme/host pattern documented (cross-reference
  `05-service-inventory.md` Step 5); `android:autoVerify="true"` requires working
  `assetlinks.json` — verify:
  `curl https://<host>/.well-known/assetlinks.json` and match the release cert SHA-256
  (`keytool -list -printcert -jarfile app-release.apk` — fingerprint only; never print keystore
  secrets).
- Overly broad filters (`scheme="http"` with no host) = hijackable surface — Medium.
- Custom scheme used for auth callbacks (OAuth) = token-interception risk; recommend App Links —
  High.

## Step 4 — Network Security

```bash
grep -n "usesCleartextTraffic\|networkSecurityConfig" "$MERGED_MANIFEST"
cat app/src/main/res/xml/network_security_config.xml 2>/dev/null
```

```powershell
Select-String -Path $MERGED_MANIFEST -Pattern "usesCleartextTraffic|networkSecurityConfig"
Get-Content app\src\main\res\xml\network_security_config.xml -ErrorAction SilentlyContinue
```

| Finding | Severity |
|---|---|
| `android:usesCleartextTraffic="true"` app-wide | High (Critical if auth/payments traffic exists) |
| `network_security_config` with `cleartextTrafficPermitted="true"` for specific domains | Medium — verify each domain still needs HTTP |
| `<trust-anchors>` adding user certs in release config | High (MITM exposure); acceptable only in debug overlay |
| No config, default behavior (cleartext blocked since API 28) | OK — note it |

## Step 5 — Backup Rules

```bash
grep -n "allowBackup\|fullBackupContent\|dataExtractionRules" "$MERGED_MANIFEST"
cat app/src/main/res/xml/backup_rules.xml app/src/main/res/xml/data_extraction_rules.xml 2>/dev/null
```

```powershell
Select-String -Path $MERGED_MANIFEST -Pattern "allowBackup|fullBackupContent|dataExtractionRules"
Get-Content app\src\main\res\xml\backup_rules.xml, app\src\main\res\xml\data_extraction_rules.xml -ErrorAction SilentlyContinue
```

- API 31+ uses `android:dataExtractionRules` (with `<cloud-backup>` and `<device-transfer>`
  sections); `android:fullBackupContent` covers API ≤ 30 — a modern app should declare **both**.
- `allowBackup="true"` (default) with no exclusion rules while the app stores auth tokens in
  files/DataStore/DB = High finding (tokens migrate to other devices, restorable by attackers
  with local access). Exclude token stores explicitly.
- `allowBackup="false"` is a blunt but acceptable fix for apps with sensitive local data; note the
  UX cost (no transfer of user settings).

## Step 6 — Task & Launch Smells

| Pattern | Risk | Severity |
|---|---|---|
| `android:taskAffinity` set to "" or custom on exported activities | Task-hijacking (StrandHogg-class) mitigation when ""; custom affinities = spoofing surface | Setting `""` on sensitive activities is the *fix*; custom shared affinities are Medium |
| `launchMode="singleTask"/"singleInstance"` on non-launcher activities | Back-stack anomalies, intent replay | Low–Medium; verify intent handling in `onNewIntent` |
| `allowTaskReparenting="true"` | Activity hijack into foreign task | Medium |
| `android:debuggable="true"` in merged **release** manifest | Full compromise | Critical |
| `testOnly`, `usesNonSdkApi` leftovers | Release blockers | High |

## Step 7 — Secrets Scan (manifest + resources)

The merged manifest and `res/` are shipped in cleartext inside every APK — anything here is
public. Scan for key/token shapes; this is the method-level home for the scan that prompt 03
orders. **Record presence, file, and line only — never copy a discovered value into any artifact**
(per the secrets rule in `02-engineering/README.md`).

```bash
# Value-bearing meta-data in the merged manifest
grep -n "meta-data" "$MERGED_MANIFEST" | grep -iE "key|token|secret"
# Known key shapes in manifest + resources (Google API key, Stripe live, GitHub PAT, AWS access key)
grep -rnE "AIza[0-9A-Za-z_-]{35}|sk_live_[0-9A-Za-z]+|ghp_[0-9A-Za-z]{36,}|AKIA[0-9A-Z]{16}" \
  app/src/main/AndroidManifest.xml app/src/main/res
# Suspicious key/token-named string resources
grep -rniE "(api[_-]?key|auth[_-]?token|client[_-]?secret|password)" app/src/main/res/values
```

```powershell
# Value-bearing meta-data in the merged manifest
Select-String -Path $MERGED_MANIFEST -Pattern "meta-data" | Select-String -Pattern "key|token|secret"
# Known key shapes in manifest + resources
Get-ChildItem -Recurse app\src\main\res -Include *.xml |
  Select-String "AIza[0-9A-Za-z_-]{35}|sk_live_[0-9A-Za-z]+|ghp_[0-9A-Za-z]{36,}|AKIA[0-9A-Z]{16}"
Select-String -Path app\src\main\AndroidManifest.xml -Pattern "AIza[0-9A-Za-z_-]{35}|sk_live_|ghp_|AKIA[0-9A-Z]{16}"
# Suspicious key/token-named string resources
Get-ChildItem -Recurse app\src\main\res\values -Filter *.xml |
  Select-String "api[_-]?key|auth[_-]?token|client[_-]?secret|password"
```

Triage every hit: **public client identifiers** (AdMob application ID, `applovin.sdk.key`,
Firebase `google_app_id` / API key in `google-services.json`-derived resources, Maps Android key
restricted by package+SHA-256) are *designed* to ship in the manifest — verify they carry the
vendor's recommended restrictions and move on. **Server-side secrets** (payment secret keys,
OAuth client secrets, bearer tokens, AWS credentials) in any shipped file = Critical finding;
rotation of the leaked credential is part of the fix, not just removal from the repo.

## Severity Guidance Summary

| Finding type | Default severity |
|---|---|
| Restricted permission without core-function justification | Critical (rejection/strike risk) |
| Exported component performing privileged work unguarded | Critical |
| `debuggable` release / user trust-anchors in release / server-side secret in shipped files | Critical |
| Cleartext app-wide; backup leaking credentials; missing `exported` explicitness; broken autoVerify | High |
| Unused dangerous permission (library-injected); broad intent filters; HTTP-whitelisted domains | Medium |
| Missing `maxSdkVersion` optimizations; launchMode quirks with verified handling | Low |

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Permission table + high-scrutiny verdicts | `docs/factory/audits/ENGINEERING_AUDIT.md` "Manifest" section; one ENG- finding per non-justified row | P4 removals; Play declaration forms at P8 (`10-releases/release-workflow.md`); Data safety form (`02-engineering/04-sdk-inventory.md` table) |
| Exported-component findings | ENG- findings | P4 hardening items; `07-checklists/release-readiness.md` |
| Network/backup/task findings | ENG- findings | P4 fixes; `15-final-verification.md` re-check |
| Secrets-scan hits (location only, never values) | ENG- findings (Critical for real secrets) | P4 removal + credential rotation; `07-checklists/release-readiness.md` |
| Merged-vs-source diff + merger attribution | ENGINEERING_AUDIT.md appendix | Library kill decisions in `02-engineering/04-sdk-inventory.md` |
| Deep-link verification results | ENGINEERING_AUDIT.md | P7 ASO campaign links; smoke flows in `02-engineering/07-build-verification.md` |

Re-run this entire method on the merged manifest after any P4 change that touches dependencies or
manifests, and once more at P8 — library updates silently change the merged result.
