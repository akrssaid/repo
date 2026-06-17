# Prompt 03 — Engineering Modernization Audit

**Lifecycle phase:** P3 — Engineering Audit & Roadmap

This prompt measures the target app against the factory's tech baseline and produces a complete,
evidence-backed engineering audit: SDK posture vs Play deadlines, deprecated API usage, dependency
staleness, build health, security smells, native-library readiness, and a performance baseline.
Every finding gets an ID, severity, effort, evidence, and recommendation, so that Prompt 04
(`01-core/prompts/04-roadmap-generation.md`) can turn the findings mechanically into a sequenced
modernization roadmap. It also creates the project's risk register — the living ledger every later
phase gate reviews.

## Role

You are a Principal Android Engineer who has led modernization programs on apps with tens of
millions of installs. You know exactly which legacy patterns rot first (kapt, AsyncTask, support
libraries, JCenter), which Play policy deadlines actually bite, and how to grade severity without
drama: Critical means "store rejection or runtime breakage is coming", not "I dislike this code".
You never report a finding without `file:line` evidence, and you separate measurement from
prescription — the roadmap (P3's second half) does the sequencing, not you.

## Objective

Produce `docs/factory/audits/ENGINEERING_AUDIT.md`, structured per
`09-templates/engineering-audit-report.md`, containing every materially relevant engineering
finding in the target repo — each with a unique ID (`ENG-001`, `ENG-002`, …), Severity
(Critical/High/Medium/Low), Effort (S/M/L/XL — scales per `01-core/CONVENTIONS.md`), evidence
(`file:line` or command output), and a concrete recommendation referencing the factory's method
docs and migration guides — plus `docs/factory/reports/RISK_REGISTER.md` instantiated from
`09-templates/risk-register.md` and seeded with the audit's risks. Done means the audit is complete
enough that Prompt 04 can build the full roadmap without re-inspecting the codebase.

## Preconditions & Required Inputs

| Requirement | Detail | If missing |
|---|---|---|
| Lifecycle phase | P3; P1 and P2 Done per the Phase Tracker. | Run `01-core/prompts/01-discovery.md` then `02-project-memory.md` first. |
| `docs/factory/audits/DISCOVERY.md` | Module map, version citations, dependency surface — your starting index. | Hard stop; re-run P1. |
| `docs/factory/PROJECT_STATE.md` | Tech Stack Snapshot and Open Questions to confirm/close. | Hard stop; re-run P2. |
| Method docs | `02-engineering/03-dependency-analysis.md`, `04-sdk-inventory.md`, `05-service-inventory.md`, `06-manifest-audit.md`, `09-r8-shrinking.md`, `10-performance-audit.md` — follow their procedures for the corresponding dimensions. | Proceed with this prompt's steps alone and note the gap in the report. |
| Baseline reference | `08-knowledge/android/version-matrix.md` — the canonical version baseline (SDK levels, AGP, Kotlin, Gradle, JDK, library floors, compatibility matrix). Grade against it; do not restate its numbers here or in the report. Also `08-knowledge/android/common-pitfalls.md`. | Hard stop for grading quality; ask the operator to fix the factory checkout. |
| Risk-register template | `09-templates/risk-register.md` — this prompt instantiates it (step 10). | Hard stop, as above. |
| Build execution | A working JDK 17 and the Gradle wrapper; ability to run `./gradlew` read-only tasks (`lint`, `:app:dependencies`). | Run the static parts of the audit; mark build-health findings `Unknown — verify` and record the blocking error verbatim. |
| Network (optional) | Version-currency checks against Maven Central / Google Maven. | Grade staleness from release knowledge as of 2026-06-10 and mark exact-latest versions `verify latest`. |
| Emulator/device (optional) | One phone profile for the performance baseline (startup-time capture, step 8). | Run the static performance checks only; mark runtime metrics `Unknown — verify`. |

## Agent Orchestration

Fan out **one subagent per audit dimension** for the read-only, parallelizable dimensions
(cap: 8 concurrent per `01-core/CONVENTIONS.md`):

| # | Dimension | Method doc | Covers procedure steps | Runs as |
|---|---|---|---|---|
| A | SDK & build-system posture | `02-engineering/04-sdk-inventory.md` | 2 | subagent |
| B | Dependency staleness | `02-engineering/03-dependency-analysis.md` | 4 | subagent |
| C | Deprecated API usage | `08-knowledge/android/common-pitfalls.md` | 3 | subagent |
| D | Manifest & security | `02-engineering/06-manifest-audit.md` | 6 | subagent |
| E | Services & SDK vendors | `02-engineering/05-service-inventory.md` | 4 (vendor SDK subset), 6 (data-safety overlap) | subagent |
| F | Native libs & 16 KB readiness | this prompt, step 7 | 7 | subagent |
| G | Performance baseline | `02-engineering/10-performance-audit.md` | 8 | **orchestrator** (device-serial) |

The orchestrator itself runs step 1 (setup), step 5's Gradle commands (build tasks must not run
concurrently from multiple agents — one Gradle daemon, one driver), step 8 (one device, one `adb`
session), and steps 9–11 (assembly, risk register, close-out). Each subagent returns findings
**only** in this exact shape, one block per finding, plus a final line `CLEAN: <dimension>` if
nothing was found:

```markdown
FINDING
Title: <one line>
Severity: Critical|High|Medium|Low
Effort: S|M|L|XL
Evidence: <file>:<line> — <quoted snippet or command output excerpt>
Detail: <2–5 lines: what, why it matters, deadline if any>
Recommendation: <concrete fix, citing factory migration guide path if one applies>
```

Subagents do not assign IDs (the orchestrator numbers `ENG-001…` after dedup) and do not write
files. For projects with ≤ 2 modules and < 200 source files, skip fan-out and run all dimensions
single-threaded in the order of the Procedure.

## Procedure

1. **Setup.** Read DISCOVERY.md and PROJECT_STATE.md. Confirm the module list and version
   citations still match reality (spot-check `gradle/wrapper/gradle-wrapper.properties` and
   `gradle/libs.versions.toml`); if discovery is stale, refresh those facts now and note it.
   Decide fan-out per Agent Orchestration. Create the findings ledger (empty list, IDs assigned at
   assembly).

2. **SDK levels vs Play deadlines** (per `02-engineering/04-sdk-inventory.md`). For every module:
   record compileSdk / targetSdk / minSdk with `file:line`. Grade against
   `08-knowledge/android/version-matrix.md`:
   - targetSdk < 35 → **Critical** (already below the current Play floor; updates blocked).
   - targetSdk = 35 → **High** (API 36 becomes the floor at the August 2026 deadline — verify the
     current Play deadline before finalizing severity).
   - compileSdk < targetSdk possible-mismatch, `<uses-sdk>` manifest overrides, per-module skew
     (modules disagreeing on minSdk) → Medium.
   - Edge-to-edge: if targetSdk ≥ 35, check for opt-outs and unmigrated
     `statusBarColor`/`navigationBarColor` usage → finding per occurrence cluster.

3. **Deprecated API usage.** Grep the target (Kotlin + Java + Gradle files) for each pattern;
   every cluster of hits becomes one finding with representative `file:line` evidence and a total
   count. Recommended `rg` patterns (escape as needed on PowerShell — prefer the Grep tool):
   - `android.os.AsyncTask|AsyncTask<` → deprecated since API 30 (not removed — grade as rot, not
     breakage); replace with coroutines (`kotlin-migration` patterns).
   - `kotlin-kapt|org.jetbrains.kotlin.kapt|\bkapt\(` in build files → migrate to KSP.
   - `getSharedPreferences|PreferenceManager.getDefaultSharedPreferences` → DataStore.
   - `startActivityForResult|onActivityResult` → Activity Result API.
   - `onBackPressed\(\)|onBackPressed\s*\{` (overrides) → OnBackPressedDispatcher / predictive back.
   - `jcenter\(\)` → remove; resolve replacements from Maven Central.
   - `com.android.support|android.support.v4|android.support.v7` → AndroidX (jetifier check:
     `android.enableJetifier` in `gradle.properties`).
   - `kotlin-android-extensions|kotlinx.android.synthetic` → ViewBinding/Compose.
   - `io.reactivex\b|rx.Observable` → note migration scope to Flow (often L/XL effort — grade
     effort honestly, do not reflexively demand it).
   - `Handler\(\)` no-Looper ctor, `MODE_WORLD_READABLE`, `requestLegacyExternalStorage`,
     `Html.fromHtml\(` single-arg — each Medium/Low as applicable.

4. **Dependency staleness** (per `02-engineering/03-dependency-analysis.md`). From the version
   catalog / build files: for each direct dependency, record declared version vs latest stable
   known (verify online when network allows; otherwise mark `verify latest`). Findings for:
   - Majors behind on critical libs (AGP, Kotlin, Compose BOM, Play Services, Billing — Billing
     library below the Play-mandated minimum is **Critical**).
   - Abandoned/archived libraries (no release in 3+ years) and known-vulnerable versions.
   - Vendor/ads/analytics SDKs (cross-check subagent E per
     `02-engineering/05-service-inventory.md`): AdMob/UMP, Firebase BoM, crash reporters —
     outdated consent/ads SDKs carry policy risk, not just tech debt.
   - Missing version catalog (versions inlined in build files) → Medium hygiene finding.

5. **Build health** (orchestrator-run; never run two Gradle invocations concurrently). Dependency
   dumps go to `docs/factory/audits/deps-<config>.txt` per the artifact-contract matrix in
   `01-core/CONVENTIONS.md`:

   ```bash
   ./gradlew --version
   ./gradlew :app:dependencies --configuration releaseRuntimeClasspath > docs/factory/audits/deps-release.txt
   ./gradlew :app:dependencies --configuration debugRuntimeClasspath  > docs/factory/audits/deps-debug.txt
   ./gradlew lint
   ```

   (Windows: `.\gradlew.bat …`. Substitute the real app module name from DISCOVERY.md.)
   - If the build fails, the failure itself is a finding (severity High+) — capture the first
     error verbatim.
   - Parse the lint report (`app/build/reports/lint-results-*.html|xml`): promote lint
     Errors to findings; summarize Warning categories with counts rather than itemizing all.
   - From the dependency dumps: duplicate-class risks, dynamic versions (`+`), forced downgrades;
     the release/debug diff reveals debug-only tooling.
   - Per `02-engineering/07-build-verification.md`, record exact build commands and durations —
     P4's verification gates will reuse them.
   - R8/resource shrinking posture: `minifyEnabled`/`shrinkResources` per build type, keep-rule
     hygiene, missing-rules risks — assess per `02-engineering/09-r8-shrinking.md` and file
     findings (release build unshrunk → Medium; broken/absent keep rules on a minified release →
     grade per that method doc).

6. **Security smells** (per `02-engineering/06-manifest-audit.md` for the manifest half — obtain
   the **merged manifest** via that doc's Step 0 first; never audit only the source manifests):
   - `android:usesCleartextTraffic="true"` or a permissive `networkSecurityConfig` → High.
   - `android:exported="true"` components without permission protection or with overly broad
     intent filters; providers with `grantUriPermissions` — judge each, cite each.
   - `android:allowBackup="true"` with sensitive data and no backup rules → Medium.
   - Hardcoded secrets: grep for `(?i)(api[_-]?key|secret|password|token)\s*[:=]\s*"[A-Za-z0-9_\-]{12,}"`,
     `AIza[0-9A-Za-z_\-]{30,}` (Google API keys), `sk_live_`, AWS `AKIA[0-9A-Z]{16}` in source,
     resources (`strings.xml`), and build files. **Report the file:line and a redacted prefix
     (first 6 chars + `…`) — never the full secret value.** Any live credential in source →
     Critical.
   - Checked-in keystores (`**/*.jks`, `**/*.keystore` in VCS), `debuggable true` on release,
     missing R8/proguard on release builds (cross-link the step-5 R8 assessment).
   - Dangerous permissions without evident feature justification (`QUERY_ALL_PACKAGES`,
     `MANAGE_EXTERNAL_STORAGE`) — Play policy rejection risk → High.

7. **16 KB page-size readiness.** Required for native code on Android 15+ devices and a Play
   requirement for targetSdk 35+ updates:
   - Detect native surface: `**/*.so` in the repo, `externalNativeBuild`/`ndkVersion` in build
     files, and AAR dependencies known to bundle natives (sqlcipher, ffmpeg, opencv, mlkit, etc.
     — check `docs/factory/audits/deps-release.txt`).
   - If an APK/AAB can be built or is present, verify alignment:
     ```bash
     # zipalign check on the APK (page alignment of .so entries)
     zipalign -c -P 16 -v 4 app-release.apk
     ```
     or inspect with `unzip -l` + `readelf -lW <lib>.so | grep LOAD` (alignment must be 0x4000).
   - No native libs at all → record explicitly as "16 KB: not applicable — no native libraries"
     (a stated non-finding, not silence). Unverifiable natives from third-party AARs → High with
     `verify with vendor` recommendation.

8. **Performance baseline** (orchestrator-run, device-serial; per
   `02-engineering/10-performance-audit.md` — the seventh audit dimension). At minimum capture
   startup time on a debug build:

   ```bash
   adb shell am force-stop <applicationId>
   adb shell am start -W -n <applicationId>/<launcherActivity>   # records ThisTime/TotalTime (cold start)
   ```

   Repeat per that method doc (multiple runs, report the median), and run its static checks
   (main-thread I/O patterns, oversized Application.onCreate init, missing baseline profiles,
   StrictMode absence). Record the startup-time numbers in the report even when they look fine —
   they are the baseline P4/P6 regressions are measured against. No device available → run the
   static checks and mark runtime metrics `Unknown — verify`.

9. **Assemble the report.** Merge all subagent findings; dedupe overlaps (e.g., an outdated ads
   SDK reported by both B and E becomes one finding with merged evidence). Assign IDs `ENG-001…`
   ordered by severity (all Criticals first), then by theme. Write
   `docs/factory/audits/ENGINEERING_AUDIT.md` following `09-templates/engineering-audit-report.md`
   exactly: executive summary (counts by severity, top 5 risks, overall grade), baseline-comparison
   table (current vs `08-knowledge/android/version-matrix.md` target per stack component), the
   findings register, the stated non-findings (dimensions checked and clean), and appendices (lint
   summary, dependency-dump paths, commands run).

10. **Instantiate the risk register.** Create `docs/factory/reports/RISK_REGISTER.md` from
    `09-templates/risk-register.md` (this prompt is its sole creator — see that template's Usage
    and `01-core/CONVENTIONS.md`). Seed it with the audit-discovered risks: every Critical/High
    finding whose danger extends beyond its own fix (Play-deadline exposure, policy-rejection
    risk, R8 release-only breakage potential, unverifiable native libs, secret exposure) becomes a
    `RISK-001…` row with category, likelihood/impact (H/M/L), exposure, mitigation pointing at the
    relevant `ENG-xxx` finding and method doc, and a named trigger. Fill the risk matrix. 3–6 seed
    risks is typical; an empty register means you have not read your own audit.

11. **Close the loop.** Verify every audit dimension (steps 2–8) is represented in the report by
    findings or an explicit `CLEAN` statement. Verify every Open Question from PROJECT_STATE.md
    that this audit could answer is either answered (with evidence) or restated. Then apply the
    Documentation Update Rules below.

## Expected Outputs

| Artifact | Path (target repo) | Notes |
|---|---|---|
| Engineering audit | `docs/factory/audits/ENGINEERING_AUDIT.md` | Structured per `09-templates/engineering-audit-report.md`; findings `ENG-001…`. |
| Dependency dumps | `docs/factory/audits/deps-release.txt`, `docs/factory/audits/deps-debug.txt` | Raw `:app:dependencies` output per config (`deps-<config>.txt` per `01-core/CONVENTIONS.md`); referenced as evidence. |
| **Risk register (created)** | `docs/factory/reports/RISK_REGISTER.md` | Instantiated from `09-templates/risk-register.md`, seeded with audit-discovered risks (step 10). Prompt 04 adds workstream risks; every phase gate reviews it. |
| Lint summary (optional) | `docs/factory/reports/lint-summary.md` | Only if lint output is too large to inline in the audit appendix. |

No source or build files in the target are modified. P3 measures; P4 plans; the migrations
(`02-engineering/migrations/`) change code.

## Acceptance Criteria

- [ ] Every finding has a unique sequential ID, Severity, Effort, `file:line` (or command-output)
      evidence, and a recommendation; zero findings rest on assertion alone.
- [ ] All seven dimensions (SDK posture, deprecated APIs, dependency staleness, build health,
      security, 16 KB readiness, performance baseline) appear in the report — each with findings
      or an explicit clean statement.
- [ ] SDK-level findings state the relevant Google Play deadline and what happens when it passes.
- [ ] Every grep pattern from step 3 was run; hit counts (including zero) are recorded.
- [ ] `./gradlew lint` and the `:app:dependencies` dumps were executed (dumps at
      `audits/deps-<config>.txt`), or their failure/unavailability is documented verbatim with
      severity assigned.
- [ ] Startup time was captured per `02-engineering/10-performance-audit.md`, or its
      unavailability is recorded as `Unknown — verify` with the reason.
- [ ] `docs/factory/reports/RISK_REGISTER.md` exists, instantiated from the template with zero
      `{{...}}` tokens, seeded with the audit's risks, matrix filled.
- [ ] No full secret values, passwords, or keystore contents appear anywhere in the report
      (redacted prefixes only).
- [ ] Severity distribution is defensible: every Critical maps to store rejection, security
      exposure, or imminent breakage — not stylistic preference.
- [ ] The report follows `09-templates/engineering-audit-report.md` section structure, cites
      `08-knowledge/android/version-matrix.md` for baseline targets instead of restating them,
      and contains zero `{{...}}` tokens.
- [ ] PROJECT_STATE.md and CHANGELOG.md updated per the rules below.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`:**
  - *Phase Tracker*: set P3 row to `In Progress` at start; after Prompt 04 also completes, P3
    becomes `Done` — this prompt alone leaves it `In Progress` with note "audit done, roadmap
    pending".
  - *Tech Stack Snapshot*: correct any value the audit proved wrong; refresh the snapshot date.
  - *Known Risks*: confirm the register link points at `docs/factory/reports/RISK_REGISTER.md`
    (now real); summarize the top seeded `RISK-xxx` rows, each annotated with its `ENG-xxx` source.
  - *Open Questions*: close every question the audit answered (cite the finding ID); add new
    `Unknown — verify` items the audit raised.
  - *Artifact Index*: add rows for ENGINEERING_AUDIT.md, `deps-release.txt`/`deps-debug.txt`, and
    RISK_REGISTER.md with today's date.
- **`docs/factory/CHANGELOG.md`:** append (top) an entry dated 2026-06-10, phase P3, summary
  "Engineering modernization audit completed; risk register created", with finding counts by
  severity and the artifact paths produced. Append-only; never rewrite prior entries.
- **`docs/factory/reports/RISK_REGISTER.md`:** created here (step 10). From now on it is reviewed
  at every phase gate per its Review Cadence section; Prompt 04 appends workstream risks next.

## Failure Modes & Recovery

1. **The build does not run** (wrong JDK, missing `local.properties` SDK path, network-locked
   plugin resolution). Do not abandon the audit: complete all static dimensions, record the exact
   build error as a High+ finding ("project does not build from a clean checkout"), and mark
   lint/dependency-dump/performance items `Unknown — verify`. A non-building project is itself a
   top audit result.
2. **Finding flood** (legacy app with 300+ raw hits). Cluster aggressively: one finding per
   pattern per architectural area with a hit count and 3 representative citations — not one
   finding per occurrence. The roadmap needs workstreams, not a phonebook.
3. **Severity inflation.** If more than ~20% of findings are Critical, re-grade against the
   definition (store rejection / security exposure / imminent breakage). Inflated audits get
   ignored; calibrated ones get funded.
4. **Subagent overlap/conflict** (B and E grade the same SDK differently). The orchestrator
   re-checks the evidence itself, keeps one merged finding with the better-supported grade, and
   notes the merge. Never publish two IDs for one issue.
5. **Version-currency uncertainty** (no network; you cannot confirm today's latest stable).
   State the declared version with citation, grade staleness from your knowledge as of 2026-06-10,
   and tag the recommendation `verify latest stable` rather than asserting a possibly-wrong target
   version.
6. **Risk-register skipped or padded.** An audit with Critical findings and an empty register, or
   a register stuffed with restated findings that carry no forward-looking danger, both fail the
   gate. A risk is an *event that may still happen* with a trigger you can watch — re-derive the
   seed set from the findings that threaten future phases, not from the findings list wholesale.
7. **Scope creep into fixing.** You will be tempted to fix one-liners (`jcenter()` removal) while
   you are there. Do not. P3 is read-only by contract; mixed audit+fix sessions corrupt the
   evidence (line numbers shift) and bypass P4's sequencing and verification gates
   (`02-engineering/07-build-verification.md`).
