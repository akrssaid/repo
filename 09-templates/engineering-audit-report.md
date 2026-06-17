# Template — Engineering Audit Report

This template produces the canonical engineering health assessment of a target Android app: build
system, dependencies, SDKs, manifest, code health, performance baseline, security, and
native-library posture. The instance is the single source of truth that
`01-core/prompts/04-roadmap-generation.md` consumes to build the modernization roadmap, so every
finding must carry evidence, severity, and effort. The same P3 prompt also **creates**
`docs/factory/reports/RISK_REGISTER.md` from `09-templates/risk-register.md`; this report's
Deferred & Accepted-Risk Log is the seed that populates it (see the handoff note in that section).

## Usage

- **Instantiated by:** the engineering auditor agent running `01-core/prompts/03-modernization-audit.md`.
- **Phase:** P3 — Engineering Audit & Roadmap.
- **Instance path:** `docs/factory/audits/ENGINEERING_AUDIT.md` in the TARGET app repo.
- **Procedure:** follow the Instantiation Doctrine in `09-templates/README.md` — copy the body
  below the marker, fill every token, delete all guidance comments and example rows, then verify
  with `grep -n "{{" docs/factory/audits/ENGINEERING_AUDIT.md` (must return nothing).

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` | Human-readable app name | Solar Camera Pro |
| `{{PACKAGE_NAME}}` | applicationId from app module | com.solarcamera.pro |
| `{{AUDIT_DATE}}` | Date audit completed, ISO 8601 | 2026-06-10 |
| `{{AUDITOR_ROLE}}` | Persona from prompt 03 Role section | Senior Android Platform Engineer |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` used for this audit | 1.2.0 |
| `{{COMMIT_SHA}}` | Full git SHA of the commit audited | 9f3c1a7e…(40 chars) |
| `{{HEALTH_VERDICT}}` | One-line overall verdict: Healthy / Aging / At Risk / Critical + qualifier | At Risk — builds, but blocked from Play target-API deadline |
| `{{CURRENT_GRADLE}}` / `{{CURRENT_AGP}}` / `{{CURRENT_KOTLIN}}` | Versions found in the repo | 7.4.2 / 7.3.1 / 1.7.21 |
| `{{CURRENT_COMPILE_SDK}}` / `{{CURRENT_TARGET_SDK}}` / `{{CURRENT_MIN_SDK}}` | SDK levels found | 33 / 33 / 21 |
| `{{CURRENT_JDK}}` | JDK/toolchain version the build currently uses | 11 |
| `{{STARTUP_COLD_MS}}` | Measured cold-start time to first frame, ms (method per `02-engineering/10-performance-audit.md`) | 1180 |
| `{{STARTUP_BAND}}` | Cold-start band vs Play vitals (Good < 500ms / Medium / Bad > 1s) per `08-knowledge/android/play-vitals-performance.md` | Bad (> 1s) |
| `{{APK_SIZE_MB}}` / `{{AAB_SIZE_MB}}` | Current release APK / AAB download size, MB | 31.2 / 27.8 |
| `{{FINDING_COUNT_CRITICAL}}` etc. | Count of findings per severity | 3 |
| `{{DIMENSION_SUMMARY_*}}` | One-paragraph verdict per dimension section (BUILD, DEPENDENCIES, SDKS, MANIFEST, CODE, PERFORMANCE, SECURITY, NATIVE) | (prose) |
| `{{ROADMAP_SEQUENCE_HINT}}` | Recommended migration order for prompt 04 | Gradle → AGP → Kotlin → SDK 36 |
| `{{ROADMAP_QUICK_WINS}}` | Comma-separated finding IDs doable in < 0.5 day each | ENG-004, ENG-011 |
| `{{ROADMAP_BLOCKERS}}` | Finding IDs that block release if unfixed | ENG-001, ENG-002 |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Engineering Audit — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Audit date | {{AUDIT_DATE}} |
| Auditor role | {{AUDITOR_ROLE}} |
| Factory version | {{FACTORY_VERSION}} |
| Commit audited | `{{COMMIT_SHA}}` |

## Executive Summary

**Health verdict:** {{HEALTH_VERDICT}}

<!-- 3–6 sentences max. State whether the app builds today, whether it can ship to Google Play
     against the current target-API deadline, and the dominant theme of debt (e.g. "toolchain
     frozen in 2022, dependencies otherwise sane"). No finding details here — verdict and shape. -->

**Findings:** {{FINDING_COUNT_CRITICAL}} Critical / {{FINDING_COUNT_HIGH}} High /
{{FINDING_COUNT_MEDIUM}} Medium / {{FINDING_COUNT_LOW}} Low.

**Top 5 risks (highest exposure first):**

<!-- One line each: finding ID + why it is dangerous in business terms, not just technical terms
     (e.g. "ENG-001 targetSdk 33 — Play will block updates after the August 2026 deadline"). -->

1. {{TOP_RISK_1}}
2. {{TOP_RISK_2}}
3. {{TOP_RISK_3}}
4. {{TOP_RISK_4}}
5. {{TOP_RISK_5}}

## Baseline Snapshot

<!-- Factory baseline column comes from 08-knowledge/android/version-matrix.md and the tech
     baseline in BEST_PRACTICES.md; verify latest stable at audit time — versions drift.
     Gap column: OK / MINOR / MAJOR / BLOCKING. -->

| Component | Current | Factory baseline (2026-06) | Gap |
|---|---|---|---|
| Gradle | {{CURRENT_GRADLE}} | 8.13+ | {{GAP_GRADLE}} |
| AGP | {{CURRENT_AGP}} | 8.9+ | {{GAP_AGP}} |
| Kotlin | {{CURRENT_KOTLIN}} | 2.1+ (K2) | {{GAP_KOTLIN}} |
| compileSdk | {{CURRENT_COMPILE_SDK}} | 36 | {{GAP_COMPILE_SDK}} |
| targetSdk | {{CURRENT_TARGET_SDK}} | 36 | {{GAP_TARGET_SDK}} |
| minSdk | {{CURRENT_MIN_SDK}} | 26 (24 acceptable) | {{GAP_MIN_SDK}} |
| JDK toolchain | {{CURRENT_JDK}} | 17 | {{GAP_JDK}} |

## Performance Baseline

<!-- Measured, not estimated. Method and instrumentation per 02-engineering/10-performance-audit.md
     (macrobenchmark or `adb shell am start -W` for cold start; bundletool/APK Analyzer for size).
     Bands for startup come from 08-knowledge/android/play-vitals-performance.md. This block feeds
     dimension 7 (Performance) of the roadmap — any row outside its band must also be a finding in
     the register with a PERF-flagged ENG-xxx ID. Record device + build type used. -->

| Metric | Current | Target / band | Source |
|---|---|---|---|
| Cold start to first frame | {{STARTUP_COLD_MS}} ms | {{STARTUP_BAND}} | `02-engineering/10-performance-audit.md` |
| APK download size | {{APK_SIZE_MB}} MB | trend, no hard cap | APK Analyzer |
| AAB / delivered size | {{AAB_SIZE_MB}} MB | trend, no hard cap | `bundletool get-size total` |

## Findings Register

<!-- One row per finding, ID format ENG-001 ascending (zero-padded 3 per CONVENTIONS.md D2),
     ordered by severity then effort.
     Dimension ∈ Build System / Dependencies / SDKs & Services / Manifest & Permissions /
     Code Health / Performance / Security / Native & 16KB. Severity = Critical/High/Medium/Low.
     Effort = S/M/L/XL. Evidence must be a real file:line or command output reference —
     a finding without evidence is an opinion and must not be filed.
     Status starts as "Not Started" for everything; prompt 04 and P4 work update it. -->

| ID | Dimension | Severity | Effort | Evidence | Recommendation | Status |
|---|---|---|---|---|---|---|
| ENG-001 | Build System | Critical | M | `app/build.gradle:14` — targetSdk 33 | Migrate to targetSdk 36 via `02-engineering/migrations/android-sdk-migration.md` | Not Started | <!-- (example — replace) -->

## Dimension Detail

<!-- Each subsection: one paragraph of summary prose, then bullets referencing finding IDs.
     Do not restate the register rows — add the context the register cannot hold: how things
     interact, what order matters, what was checked and found CLEAN (absence of findings must
     be explicit, e.g. "no kapt usage found"). -->

### Build System

{{DIMENSION_SUMMARY_BUILD}}

### Dependencies

{{DIMENSION_SUMMARY_DEPENDENCIES}}

<!-- Include: version-catalog adoption status, count of dependencies more than 2 major versions
     behind, any abandoned libraries (no release in 24+ months), kapt→KSP candidates. -->

### SDKs & Services

{{DIMENSION_SUMMARY_SDKS}}

<!-- Third-party SDKs (ads, analytics, crash), Google services, backend endpoints. Flag any SDK
     below its provider's minimum supported version and any with known policy issues per
     08-knowledge/stores/policy-landmines.md. -->

### Manifest & Permissions

{{DIMENSION_SUMMARY_MANIFEST}}

<!-- Every dangerous permission must be justified or filed as a finding. Note exported
     components, missing android:enableOnBackInvokedCallback, edge-to-edge readiness for SDK 35+. -->

### Code Health

{{DIMENSION_SUMMARY_CODE}}

<!-- Java/Kotlin ratio, AsyncTask/RxJava remnants, SharedPreferences vs DataStore, deprecated
     API usage counts from a real lint run. -->

### Performance

{{DIMENSION_SUMMARY_PERFORMANCE}}

<!-- Interpret the Performance Baseline block: cold-start band, startup regressions from heavy
     Application.onCreate work, main-thread I/O, oversized assets driving APK/AAB size, missing R8.
     Method per 02-engineering/10-performance-audit.md; bands per
     08-knowledge/android/play-vitals-performance.md. If not measured, say so and file a finding. -->

### Security

{{DIMENSION_SUMMARY_SECURITY}}

<!-- Cleartext traffic config, hardcoded secrets scan result, signing config exposure,
     backup rules, ProGuard/R8 status. -->

### Native & 16 KB Page Size

{{DIMENSION_SUMMARY_NATIVE}}

<!-- List every .so and its 16 KB alignment status (`check_elf_alignment` or
     `objdump -p lib.so | grep LOAD`). If no native libs: state that explicitly. -->

## Deferred & Accepted-Risk Log

<!-- Findings the owner explicitly chose not to fix this cycle. Each entry needs a reason and a
     revisit trigger. HANDOFF: this P3 prompt CREATES docs/factory/reports/RISK_REGISTER.md from
     09-templates/risk-register.md, seeded from the rows below — every row here must land as a RISK
     row there (with a RISK-NNN id), and that register surfaces into PROJECT_STATE.md's "Known
     Risks" section (D6 canonical name). Empty is valid — write "None as of {{AUDIT_DATE}}". -->

| Finding | Reason deferred | Revisit trigger | Risk register ref |
|---|---|---|---|
| ENG-009 minSdk 21 | 4% of installs still on API 21–23 | Installed-base share < 2% | RISK-003 | <!-- (example — replace) -->

## Inputs for Roadmap (Prompt 04 handoff)

<!-- This section is what 01-core/prompts/04-roadmap-generation.md reads first. Be directive. -->

- **Recommended migration sequence:** {{ROADMAP_SEQUENCE_HINT}}
- **Quick wins (ship independently, any order):** {{ROADMAP_QUICK_WINS}}
- **Release blockers (roadmap must front-load):** {{ROADMAP_BLOCKERS}}
- **Constraints discovered:** <!-- e.g. "AGP 8.x requires Gradle 8.x first; Kotlin 2.1 breaks
  library X pinned at Y — see ENG-007". List every ordering constraint as a bullet. -->
