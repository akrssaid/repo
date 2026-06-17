# Release Workflow — Target Apps

This is the operational procedure for shipping a TARGET app release, from a confirmed
verification PASS to post-release close-out. It executes the package prepared by
`01-core/prompts/16-release-preparation.md` and consumes
`docs/factory/handovers/RELEASE_HANDOVER_v<N>.md`. Stages are sequential; a stage may not begin
until the previous stage's exit criteria are met. The agent prepares and verifies; the human
signs, uploads, and promotes — see the swimlane at the end.

> **Authoritative rollout-gate source.** Stage 4 below is the SINGLE SOURCE for every rollout
> stage, dwell, and threshold number in the factory. Prompt 16 and
> `05-handover/RELEASE_HANDOVER_SCHEMA.md` carry no rollout table of their own — they execute
> stages 4–5 by reference. The underlying Play bad-behavior thresholds and startup bands are
> owned by `08-knowledge/android/play-vitals-performance.md`; stage 4 cites them, never restates
> them. Conventions (tags, branches, verdict vocabulary, paths) come from `01-core/CONVENTIONS.md`.

## Stage overview

| Stage | Name | Owner | Exit criterion |
|---|---|---|---|
| 1 | Pre-flight | Agent + Human sign-off | Readiness checklist 100%, version locked, tag pushed |
| 2 | Build & Sign | Agent builds, Human signs | Signed AAB + checksum + mapping.txt archived |
| 3 | Store Submission | Human uploads, Agent preps | Build live on Play internal track |
| 4 | Staged Rollout | Human promotes, Agent monitors | 100% on Play, other stores submitted |
| 5 | Monitoring | Agent | 72h watch complete, no open alerts |
| 6 | Rollback (contingency) | Human decides, Agent executes prep | Only entered on alert |
| 7 | Close-out | Agent | Release report done, state docs updated |

## 1. Pre-flight

1. **Confirm verification PASS.** Open `docs/factory/reports/VERIFICATION_REPORT_v<versionName>.md`
   produced by `01-core/prompts/15-final-verification.md` (re-runs append `_r2`, `_r3`). The
   verdict must be **PASS** or **PASS with waivers** with every waiver named and accepted in the
   handover. **FAIL** → stop; return to P8 start. The verdict vocabulary is PASS / PASS with
   waivers / FAIL per `01-core/CONVENTIONS.md`; "CONDITIONAL" does not exist. Do not reinterpret
   an unwaived gap as a note.
2. **Validate the handover.** Confirm `RELEASE_HANDOVER_v<N>.md` conforms to
   `05-handover/RELEASE_HANDOVER_SCHEMA.md` and has passed
   `07-checklists/handover-validation.md`. It must contain the rollback/forward-fix plan —
   absence blocks the release (release doctrine, `10-releases/README.md`).
3. **Run the readiness checklist.** Execute `07-checklists/release-readiness.md` item by item.
   Record the completed checklist in the handover. One unchecked item = blocked release.
4. **Lock the version.** Apply `10-releases/app-versioning-policy.md`: decide the semver bump
   from the change set, compute the `versionCode`, and verify it has never been used on any
   store (check the **Release History** section of `docs/factory/PROJECT_STATE.md` and the Play
   Console).
5. **Tag and branch.** Releases cut from the default branch at a green commit. App release tags
   are `release/vX.Y.Z` per `01-core/CONVENTIONS.md`:

   ```bash
   git checkout main && git pull --ff-only
   git tag -a release/v2.3.0 -m "Release 2.3.0 (versionCode 2030001)"
   git push origin release/v2.3.0
   ```

   ```powershell
   git checkout main; git pull --ff-only
   git tag -a release/v2.3.0 -m "Release 2.3.0 (versionCode 2030001)"
   git push origin release/v2.3.0
   ```

   Branch policy: no long-lived release branches by default. Create `hotfix/v2.3.1` from the tag
   only when a forward-fix must ship while `main` has unreleased work. Everything that ships
   must be reachable from a `release/vX.Y.Z` tag.
6. **Human sign-off.** The human confirms: version, tag, changeset summary, rollback plan. This
   sign-off is recorded in the handover. Exit stage 1.

## 2. Build & Sign

1. **Clean release build** from the tagged commit:

   ```bash
   git checkout release/v2.3.0
   ./gradlew clean bundleRelease
   ```

   ```powershell
   git checkout release/v2.3.0
   .\gradlew clean bundleRelease
   ```

   Output: `app/build/outputs/bundle/release/app-release.aab`. Build per-store flavors if the
   project defines them (e.g. `bundlePlayRelease`, `bundleRustoreRelease`).
2. **Keystore doctrine — human-held, always.** The agent never reads, copies, or references the
   keystore file, store passwords, or key passwords; they must not appear in `gradle.properties`
   committed to the repo, in CI logs, or in any factory artifact. Signing happens one of two
   ways: (a) the human runs the `bundleRelease` step on a machine where signing config resolves
   from their local untracked `keystore.properties`, or (b) Play App Signing handles the release
   key and the human uploads an upload-key-signed bundle. If the agent encounters credentials in
   the repo, it stops and reports a security finding — it does not proceed.
3. **Checksum and archive.** Record evidence before anything is uploaded:

   ```powershell
   Get-FileHash app\build\outputs\bundle\release\app-release.aab -Algorithm SHA256
   ```

   ```bash
   sha256sum app/build/outputs/bundle/release/app-release.aab
   ```

   Archive together, referenced from the handover: the AAB SHA-256, the git tag + commit SHA,
   and `app/build/outputs/mapping/release/mapping.txt` (mandatory — without it, post-release
   crash stacks are unreadable). Keep `mapping.txt` per versionCode forever; Play accepts it as
   a deobfuscation-file upload, but the local archive is the source of truth.
4. **Sanity-check the artifact**: `versionCode`/`versionName` inside the bundle match the lock
   from stage 1 (`aapt2 dump badging` on a derived universal APK via bundletool, or check the
   Play Console pre-review after upload). Exit stage 2.

## 3. Store Submission

**Doctrine: Play-first canary** (per `08-knowledge/stores/store-comparison.md`). Google Play has
the best staged-rollout mechanics and vitals telemetry, so it absorbs risk first. AppGallery and
RuStore receive the build only after Play production reaches 50% with clean vitals.

### 3a. Google Play — track promotion path

| Track | Audience | Purpose | Dwell |
|---|---|---|---|
| Internal testing | Owner's devices (≤100 testers) | Install sanity, signing check, smoke test on real hardware | Hours |
| Closed testing | Small opt-in cohort | Pre-release vitals signal, upgrade-path test from previous version | 1–3 days |
| Production (staged) | Public, percentage-gated | The actual release | Per stage 4 |

1. Human uploads the AAB to **Internal testing** with the release notes prepared by prompt 16.
2. Agent confirms via testers: fresh install, upgrade-over-previous-version install (critical —
   this is where DB migrations break; run the upgrade-path install test from
   `02-engineering/11-data-migration-safety.md`), core flows from the verification report's smoke
   list.
3. Human promotes the same build internal → closed (skip closed only for PATCH releases with
   trivial diffs). After dwell with no new crash clusters, promote to **Production** at 10%.

### 3b. Huawei AppGallery and RuStore — sequencing

1. Submit to AppGallery and RuStore only after the Play canary gate (50% clean — see stage 4).
2. Same versionCode and bundle/APK lineage everywhere; store-flavor `versionName` suffixes per
   `10-releases/app-versioning-policy.md`. Listing text comes from the ASO artifacts (P7);
   per-store quirks (review SLAs, required docs, APK vs AAB) per `04-aso/stores/huawei-appgallery.md`
   and `04-aso/stores/rustore.md`.
3. Neither store has Play-grade staged rollout; the Play canary IS their de-risking. Treat their
   review approval date as their release date in the release report.

## 4. Staged Rollout (Google Play) — authoritative rollout-gate source

**This stage is the SINGLE SOURCE for all rollout stage, dwell, and threshold numbers.** Prompt
16 and the RELEASE handover schema defer here and restate nothing. The underlying Play
bad-behavior thresholds these gates derive from — **1.09%** user-perceived crash rate and
**0.47%** user-perceived ANR rate — plus the startup-time bands are owned by
`08-knowledge/android/play-vitals-performance.md`; this stage holds the *gate* numbers and cites
that file for the *bad-behavior* numbers they sit below.

**Stages and minimum dwell.** Production starts at **10%** and climbs **10% → 25% → 50% → 100%**,
with a **minimum 24 h dwell at every stage** (the agent may recommend a longer dwell; never a
shorter one).

| Gate | Minimum dwell | Promote only if (all true) |
|---|---|---|
| 10% → 25% | 24 h | No HOLD condition active; crash-free sessions ≥ previous release baseline |
| 25% → 50% | 24 h | Above, plus no new crash cluster in the top 5 by volume |
| 50% → 100% | 24 h | Above, plus ratings velocity not degraded vs trailing 7 days |

**HOLD conditions** — any one freezes the rollout at the current percentage immediately:

- **Crash:** user-perceived crash rate exceeds the previous release's baseline by **+0.2 pp** or
  more. (The absolute Play bad-behavior line is **1.09%**; the +0.2 pp gate trips well before it —
  do not ride up to the line. Basis: `08-knowledge/android/play-vitals-performance.md`.)
- **ANR:** user-perceived ANR rate reaches **≥ 0.40%**. (The Play bad-behavior line is **0.47%**;
  HOLD at 0.40% leaves margin. Basis: same vitals doc.)
- Any new crash cluster touching data writes, payments, or login.
- Review keyword scan (stage 5) shows a repeating new-bug complaint (≥ 3 independent mentions).

**Decision table** (one owner per decision; the agent never changes rollout percentages):

| Situation | Decision | Owner | Action |
|---|---|---|---|
| All gates green at dwell end | Advance to next % | Human (agent recommends) | Human raises % in console |
| HOLD condition, cause unknown | Hold | Agent flags → Human confirms | Freeze %, agent investigates ≥ 24 h |
| HOLD condition, fix identified, low severity | Hold + forward-fix | Human | Patch release via stage 6 fast path |
| Crash/ANR severe or active data loss | Rollback (halt + forward-fix) | Human, immediately | Stage 6 in full |
| Gates green but external risk (holiday, infra freeze) | Discretionary hold | Human | Document reason in handover |

At every gate the agent produces a written **advance / hold / rollback** recommendation with vitals
evidence, recorded in the release report; the human owns the console action.

## 5. Monitoring — first-72h watch list

Per store, from the moment users can install. Agent runs the watch and reports at 24/48/72 h.

| Signal | Source | Alert threshold |
|---|---|---|
| Crash rate (user-perceived) | Play vitals / AppGallery quality / RuStore stats | > baseline +0.2 pp → HOLD (bad-behavior line 1.09%, see vitals doc) |
| ANR rate | Play vitals | ≥ 0.40% → HOLD; ≥ 0.47% (bad-behavior line, vitals doc) → halt |
| New crash clusters | Crash reporting (deobfuscated via archived mapping.txt) | Any cluster in top 5 within 24 h → investigate |
| Ratings velocity | Store consoles | 1–2★ share > 2× trailing 7-day average → investigate |
| Review keyword scan | New reviews, all stores | ≥3 independent mentions of the same new symptom → HOLD |
| Install/update failure | Console install stats | Visible spike vs previous release → investigate |

All threshold numbers in this table are the stage 4 gate numbers; the bad-behavior lines they cite
(1.09% crash, 0.47% ANR) and startup bands live in `08-knowledge/android/play-vitals-performance.md`.
Any HOLD/halt finding routes back into the stage 4 decision table; severe findings open stage 6.

## 6. Rollback Procedure

**Play cannot unship.** Once a user has installed versionCode N, the console cannot push N−1 to
them. "Rollback" on Play means: **halt the staged rollout** (stops new users from receiving the
bad build) plus a **forward-fix** release for everyone who already has it.

1. **Halt** (human): set the rollout to halted in the Play Console. Existing installs keep the
   bad build; no new ones receive it.
2. **Forward-fix fast path** (emergency release): minimal-diff fix branched from the release tag
   (`hotfix/v2.3.1`), PATCH bump + new versionCode per the versioning policy, abbreviated
   verification — run the smoke subset of `01-core/prompts/15-final-verification.md` scoped to
   the fix plus the regression that triggered it, full `07-checklists/release-readiness.md`
   still applies. Ship internal → production; staged at 50% minimum unless data loss is active,
   then 100% with human sign-off recorded.
3. **Schema-migration pre-flight — before any DB-schema release, not after.** Because Play cannot
   unship (above), a schema release that breaks the upgrade path can only be repaired by a
   forward-fix, so the reversibility question is answered in **stage 1**, not here: if version N
   migrates the schema, can N−1's code still open the DB? (It cannot, for destructive migrations.)
   Therefore any release containing a schema migration must, per
   `02-engineering/11-data-migration-safety.md`, (a) carry a forward-fix-only rollback plan in the
   handover, (b) pass the **upgrade-path install test** — install the previous store version, seed
   real data, install version N over it, confirm the migration runs clean and no data is lost —
   and (c) verify exports/backups. A schema release without this pre-flight fails stage 1.
4. **Other stores**: if the bad build already shipped to AppGallery/RuStore, submit the
   forward-fix there immediately (no canary wait — the fix is the canary's graduate). If it had
   not shipped yet (Play-first doctrine working as intended), simply submit the fixed build.
5. **Record everything**: trigger, timeline, decision, fix versionCode — in the release report
   and `docs/factory/CHANGELOG.md`.

## 7. Close-out

1. Instantiate `09-templates/release-report.md` into
   `docs/factory/reports/RELEASE_REPORT_v<versionName>.md` and complete it: versions, dates per
   store, rollout timeline with gate evidence, vitals before/after, incidents, decisions.
2. Finalize `RELEASE_HANDOVER_v<N>.md`: mark consumed, link the release report and the
   `VERIFICATION_REPORT_v<versionName>.md`.
3. Update `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md` (release entry dated per
   store go-live). In PROJECT_STATE, write **both** canonical sections per
   `01-core/CONVENTIONS.md`:
   - **Release History** — versionName/versionCode, date, tag, per-store status, rollout outcome.
   - **Artifact Index** — rows for the release report and verification report (artifact, path,
     version, date, status).
4. Write retro notes in the release report: what slowed the release, which gate data was hard to
   get, any factory-level fix → propose it and log it in `10-releases/FRAMEWORK_CHANGELOG.md`.
5. Confirm archives: tag pushed, AAB checksum recorded, mapping.txt stored. Release is closed.

## Swimlane — agent vs human

```
STAGE          AGENT                                   HUMAN
─────────────  ──────────────────────────────────────  ─────────────────────────────────
1 Pre-flight   Verify PASS, run checklist,             Review + sign off version, tag,
               compute version, draft tag cmds  ────►  rollback plan; push tag
2 Build/Sign   Run bundleRelease (unsigned path),      Hold keystore, produce signed
               compute SHA-256, archive mapping  ───►  AAB (or upload-key sign)
3 Submission   Prepare release notes, listing          Upload AAB, create releases,
               text, track plan               ──────►  promote internal → closed → prod
4 Rollout      Watch vitals, write promote/hold        Click promote/hold/halt in
               recommendation at each gate    ──────►  console; own the decision
5 Monitoring   24/48/72h reports, keyword scan,        Acknowledge reports; call for
               deobfuscate crashes            ──────►  stage 6 if severe
6 Rollback     Prepare hotfix branch, fix, fast-path   Halt rollout, sign + upload
               verification, new versionCode  ──────►  forward-fix
7 Close-out    Release report, PROJECT_STATE,          Review retro; approve factory
               CHANGELOG, retro draft         ──────►  changelog entries
─────────────  ──────────────────────────────────────  ─────────────────────────────────
RULE: credentials, console sessions, and rollout buttons are human-only, always.
```
