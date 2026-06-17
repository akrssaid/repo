# Release Readiness Checklist

Gate for **P8 (Verification & Release)** — the final gate before store submission. It is consumed
by `01-core/prompts/15-final-verification.md` (which executes it and produces the verification
report) and its PASS is a precondition for `01-core/prompts/16-release-preparation.md`. It
verifies the release artifact itself, end-to-end quality on the actual release build, store
compliance, and that every release-supporting artifact (notes, rollout plan, rollback plan,
handover) exists and is current. Nothing ships on a FAIL — there is no "ship now, fix in next
release" path through this gate.

> **Run when:** P8, after the release candidate is built and `07-checklists/accessibility.md` and
> `07-checklists/aso-launch.md` have passed.
> **Run by:** The verification agent running `01-core/prompts/15-final-verification.md`; the
> signing-key confirmation requires the human owner.
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification.
> Record the completed copy as `docs/factory/reports/release-readiness-YYYY-MM-DD.md`.

All commands run from the target app repo root. "Release build" = the exact AAB/APK to be
uploaded, not a debug or earlier build.

## Build

- [ ] Clean release bundle builds from a clean checkout state. Verify:
      `./gradlew clean :app:bundleRelease` exits 0; artifact exists at
      `app/build/outputs/bundle/release/app-release.aab`.
- [ ] R8/ProGuard mapping file archived alongside release records. Verify:
      `app/build/outputs/mapping/release/mapping.txt` exists and is copied to
      `docs/factory/reports/release-v<versionName>/mapping.txt` (uploaded for deobfuscated
      crash reports at publish time).
- [ ] Bundle is signed with the **release** key — confirmed by the human owner (key access is
      human-held). Verify signature identity:
      `keytool -printcert -jarfile app-release.aab` (or `apksigner verify --print-certs` on the
      universal APK) and have the owner confirm the SHA-256 matches the production key. Human
      confirmation recorded: name/date = ____.
- [ ] `versionCode` incremented above the last published release and consistent across
      `app/build.gradle(.kts)`, `docs/factory/PROJECT_STATE.md` (release tracker), and the
      release notes header. Verify: grep all three; values identical. Policy:
      `10-releases/app-versioning-policy.md`.

## Quality

- [ ] Unit tests green on the release variant. Verify: `./gradlew :app:testReleaseUnitTest`
      exits 0; record test count.
- [ ] Lint clean or baselined with the baseline count unchanged or reduced versus the P4 gate
      report. Verify: `./gradlew :app:lintRelease` exits 0; compare counts against
      `docs/factory/reports/engineering-modernization-*.md`.
- [ ] Crash-free smoke on the **release build** (R8-processed — debug passes prove nothing about
      shrinker breakage): install, cold start, top-5 traffic flows, and every monetization
      surface (ad load/show, paywall/purchase flow in test mode). Verify:
      `adb install -r <release-apk>` then the scripted pass;
      `adb logcat -d *:E | rg "FATAL|MissingClass|ClassNotFoundException"` returns nothing.
- [ ] No debug logging leakage in the release build — `-assumenosideeffects` log-stripping
      verified per `02-engineering/09-r8-shrinking.md`. Verify: confirm the
      `-assumenosideeffects class android.util.Log { … }` rule is present in `proguard-rules.pro`
      (or the equivalent wrap-guard the doc prescribes), then prove it took effect at runtime:
      `rg "Log\.(d|v)\(" --type kotlin --type java app/src/main` shows call sites in source, while
      `adb logcat -d | rg "<app-tag>"` during the release-build smoke shows no debug/verbose spam.
- [ ] No test/staging endpoints, test ad unit IDs, or debug feature flags in the release config.
      Verify: `rg "staging|sandbox|test\.|10\.0\.2\.2|ca-app-pub-3940256099942544" app/src/main`
      and inspect `buildConfigField` values in the release build type — only production values.
- [ ] Upgrade-path install test passed: installing this build **over** the last published version
      preserves user data and does not crash on first launch (no destructive migration). Verify per
      `02-engineering/11-data-migration-safety.md`: install the previously published APK, generate
      app data (DB/prefs/files), then `adb install -r <release-apk>` over it, launch, confirm data
      intact and no crash; `adb logcat -d *:E | rg "FATAL|SQLite|IllegalStateException"` empty.
- [ ] Pseudo-locale sweep on the release build: top flows usable under the accented
      (`en-XA`) and RTL (`ar-XB`) pseudo-locales — no truncation, no missing translations, no
      layout breakage. Verify: `adb shell setprop persist.sys.locale en-XA` (then `ar-XB`), or set
      the pseudo-locale in Developer options, walk the top-5 flows, screenshot, restore.

## Compliance

- [ ] `targetSdk` meets the current Google Play target-API requirement (36 per the June 2026
      baseline; deadline cycles every August — verify the live requirement before judging).
      Verify: grep `targetSdk`; compare with the current Play policy.
- [ ] Every permission in the merged manifest is justified in `docs/factory/PROJECT_STATE.md`
      (permission → feature mapping), with no orphaned permissions from removed features. Verify:
      `./gradlew :app:processReleaseMainManifest` then inspect
      `app/build/intermediates/merged_manifests/release/AndroidManifest.xml` `uses-permission`
      entries against the state doc table (audit method: `02-engineering/06-manifest-audit.md`).
- [ ] Data safety declaration is current for **this** build: no SDK added/removed since the
      `07-checklists/aso-launch.md` run, or the form was updated and the aso-launch Policy group
      re-run. Verify: diff dependency list against the SDK inventory snapshot date.
- [ ] All applicable Play declaration forms are filed for features the app actually uses:
      foreground-service `specialUse` declaration, exact-alarm (`USE_EXACT_ALARM`/
      `SCHEDULE_EXACT_ALARM`) justification, and any restricted/sensitive permission declarations
      (e.g. `MANAGE_EXTERNAL_STORAGE`, accessibility, SMS/Call Log). Verify: for each such
      permission present in the merged manifest, the corresponding Play Console declaration is
      drafted/filed; mark N/A per form with justification when the app does not use that feature.
- [ ] Pre-launch report reviewed (Play Console pre-launch / Firebase Test Lab on the uploaded
      bundle): crashes, security, and accessibility findings triaged. Verify: the pre-launch report
      for this `versionCode` has been read and every flagged issue is fixed, waived with rationale,
      or filed as a follow-up ticket; record the link/path. N/A only for stores without the feature,
      with justification.

## Artifacts

- [ ] VERIFICATION_REPORT with overall verdict PASS exists at
      `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md` (produced by
      `01-core/prompts/15-final-verification.md`, template `09-templates/verification-report.md`;
      re-runs append `_r2`, `_r3`). Verify: `Test-Path` and verdict field = PASS (or
      PASS with waivers, with the waivers enumerated).
- [ ] Release notes ready for every claimed store locale, within per-store length limits (Play:
      500 chars per locale). Verify: locale list matches the ASO handover; count each with
      PowerShell `("<text>").Length`.
- [ ] Rollout plan recorded: staged percentage steps, dwell time per step, and the vitals
      thresholds that halt promotion (e.g. user-perceived crash rate < 1.09%, ANR rate < 0.47% —
      Play core vitals bad-behavior thresholds). Verify: plan section exists in the RELEASE
      handover with explicit numbers, per `10-releases/release-workflow.md`.
- [ ] Rollback plan concrete: halt-rollout steps, what a fixed-forward release requires (new
      `versionCode` — binaries cannot be re-published), and who decides. Verify: section exists
      and steps are executable without questions.

## State

- [ ] `docs/factory/PROJECT_STATE.md` phase tracker shows P8 In Progress with P1–P7 Done, and the
      release tracker row for this version is filled. Verify: read the file.
- [ ] `docs/factory/CHANGELOG.md` has the release entry for `v<versionName> (<versionCode>)`
      dated today, summarizing user-facing changes. Verify: read the changelog.
- [ ] `docs/factory/handovers/RELEASE_HANDOVER_v<N>.md` validates against
      `05-handover/RELEASE_HANDOVER_SCHEMA.md` and has a PASS report from
      `07-checklists/handover-validation.md` in `docs/factory/reports/`.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / PASS with waivers / FAIL |
| Waivers (if PASS with waivers) | (item + waiver rationale + owner, or "none") |
| Release version | vX.Y.Z (versionCode N) |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |
| Human signing confirmation | (owner name + date) |

On PASS (or PASS with waivers — every waiver recorded in the Sign-off block with rationale and
owner): proceed to `01-core/prompts/16-release-preparation.md` and log this report in
`docs/factory/CHANGELOG.md`. On FAIL: fix, rebuild the release candidate if code changed, and
re-run — a rebuilt artifact invalidates the Build and Quality groups entirely.
