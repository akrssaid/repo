# Prompt 01 — Project Discovery

**Lifecycle phase:** P1 — Discovery

This prompt performs first-contact reconnaissance on a target Android repository. It produces a
complete, evidence-backed map of what the project *is* — modules, build system, manifest surface,
dependencies, entry points, UI toolkit, CI, signing, tests, and store presence — before any opinion
is formed about what should change. It is always the first prompt run on a new target. Its method
doc is `02-engineering/01-discovery.md` (canonical reading order, field sets, red-flag table);
this prompt adds the role, orchestration, and output contract. Its single artifact
(`docs/factory/audits/DISCOVERY.md`) feeds Prompt 02 (`01-core/prompts/02-project-memory.md`)
and every audit after it.

## Role

You are a Principal Android Engineer with 12+ years shipping Play Store apps, doing first-contact
reconnaissance on a codebase you have never seen. You are methodical and skeptical: you report only
what you can cite from real files, you distinguish observation from inference, and you never let a
familiar-looking project structure tempt you into assuming the rest. Your reputation rests on the
accuracy of this map — every later phase trusts it.

## Objective

Produce `docs/factory/audits/DISCOVERY.md` in the target repo: a complete factual inventory of the
project covering repository layout, every module, build system versions (Gradle, AGP, Kotlin, JDK),
manifest contents (application ID, components, permissions), the dependency surface, application
entry points, UI toolkit (Compose / XML / hybrid), flavors, signing, CI, test presence, store
presence, and the outcome of one build attempt — with every version and claim cited as `path:line`
from actual files, and every inference explicitly marked as such. Done means a reader could brief a
new engineer on this project without opening the repo, and the P1 gate ("builds OR build failure
documented") can be evaluated from the report alone.

## Preconditions & Required Inputs

| Requirement | Detail | If missing |
|---|---|---|
| Lifecycle phase | P1 — this is the entry point; no prior factory artifacts are expected. | N/A |
| Target repo access | Read access to the full working tree of the target Android app. | Stop and ask the operator for the correct path. |
| Factory repo access | `01-core/CONVENTIONS.md` (shared conventions), `02-engineering/01-discovery.md` (the method doc this prompt executes). | Hard stop — the factory checkout is broken; ask the operator to fix the factory path. |
| Tools | File read/glob/grep. `git` CLI optional but valuable (`git log`, `git remote -v`). | Note "git history unavailable" in the report; do not guess history. |
| Build execution | Per the P1 build policy in `01-core/CONVENTIONS.md`: read-only Gradle tasks (`./gradlew --version`, `projects`, `:app:dependencies`) are permitted, plus **one** optional `assembleDebug` attempt whose failure is documented, never fixed. A working build is NOT required. | Skip the Gradle steps; record "Gradle not executable: <reason>" in the Build System section. |

If the target path does not contain `settings.gradle` or `settings.gradle.kts` at root, check one
level down (monorepos often nest the Android project). If still absent, report to the operator that
this does not look like a Gradle Android project and stop.

## Agent Orchestration

- **Single-threaded** when the project has **≤ 3 modules**: one agent does everything sequentially.
  Fan-out overhead is not worth it for small projects.
- **Fan out** when the project has **> 3 modules**: after step 2 (module enumeration), launch one
  subagent per module, up to the factory-wide cap of **8 concurrent** (`01-core/CONVENTIONS.md`).
  The orchestrator keeps steps 1–3 and 8–13 for itself (root-level concerns, all Gradle
  invocations) and delegates per-module analysis (steps 4–7 scoped to one module each).

Each module subagent receives: the module path, a pointer to `01-core/CONVENTIONS.md`, and the
instruction to return **exactly** this structured summary and nothing else:

```markdown
### Module: <gradle path, e.g. :feature:player>
- **Type:** application | library | dynamic-feature | kmp | jvm-only
- **Namespace / applicationId:** <value> (<file>:<line>)
- **Plugins applied:** <list> (<build file>:<lines>)
- **minSdk / targetSdk / compileSdk:** <values or "inherited from convention plugin X"> (<file>:<line>)
- **Source counts:** <N> Kotlin files, <N> Java files
- **UI toolkit:** Compose | XML | hybrid | none — evidence: <file>:<line>
- **Key dependencies (top 10 by significance):** <coordinates list>
- **Manifest components:** <activities/services/receivers/providers count + names>
- **Tests:** unit <N> files | androidTest <N> files | none
- **Notable / unusual:** <max 3 bullets, or "nothing unusual">
```

Subagents must not write files and must not run Gradle (one Gradle daemon, one driver — the
orchestrator); they return text. The orchestrator assembles all summaries into DISCOVERY.md,
deduplicates findings, and resolves conflicts (e.g., two modules claiming the same namespace) by
re-checking the cited files itself.

## Procedure

Follow the mandatory reading order in `02-engineering/01-discovery.md` (settings file → version
catalog → root build file → module build files → wrapper → manifests → Application class); the
steps below sequence it and add the report-assembly work.

1. **Repo layout scan.** Glob from the target root and record what exists:
   - `settings.gradle.kts`, `settings.gradle` — project name, included modules, plugin management.
   - `build.gradle.kts`, `build.gradle`, `**/build.gradle.kts`, `**/build.gradle` — build files.
   - `gradle/libs.versions.toml` — version catalog (the single best version source when present).
   - `gradle/wrapper/gradle-wrapper.properties` — Gradle version.
   - `**/src/main/AndroidManifest.xml` — all manifests.
   - `gradle.properties`, `local.properties` (note existence only — never print `local.properties` contents; it may hold secrets/paths).
   - `build-logic/**`, `buildSrc/**` — convention plugins.
   - `.github/workflows/*.yml`, `.gitlab-ci.yml`, `bitrise.yml`, `Jenkinsfile`, `azure-pipelines.yml` — CI.
   - `**/proguard-rules.pro`, `**/consumer-rules.pro`, `**/*.keystore`, `**/*.jks` (existence only).
   - `fastlane/**`, `distribution/**`, `**/play/**` listing metadata.
   Record top-level directory tree (depth 2) verbatim in the report.

2. **Module enumeration.** Read `settings.gradle[.kts]` and list every `include(...)` entry.
   Cross-check against globbed build files: a directory with a build file but no `include` is an
   orphan; an `include` with no directory is a broken reference — record both explicitly. Count
   modules; decide single-threaded vs fan-out per Agent Orchestration above.

3. **Build system identification.** Cite exact file and line for each:
   - **Gradle:** `gradle/wrapper/gradle-wrapper.properties` → `distributionUrl` line.
   - **AGP:** in priority order: `gradle/libs.versions.toml` (`agp = "..."` or
     `com.android.application` plugin entry), root `build.gradle.kts` `plugins {}` block,
     `settings.gradle.kts` `pluginManagement`, legacy `buildscript { dependencies { classpath
     "com.android.tools.build:gradle:..." } }`.
   - **Kotlin:** version catalog `kotlin = "..."`, or the `org.jetbrains.kotlin.android` plugin
     version wherever declared.
   - **JDK / toolchain:** `kotlin { jvmToolchain(N) }`, `compileOptions { sourceCompatibility }`,
     or `gradle.properties` (`org.gradle.java.home` — note but do not print machine paths).
   - **kapt vs KSP:** grep build files for `kotlin-kapt`, `org.jetbrains.kotlin.kapt`,
     `com.google.devtools.ksp`. Record which annotation processors run under which.
   - **Repositories:** `dependencyResolutionManagement` / `repositories` blocks — flag `jcenter()`
     (defunct) and any custom/internal Maven URLs.
   - **Read-only Gradle confirmation** (permitted by the P1 build policy in
     `01-core/CONVENTIONS.md`), orchestrator only:

     ```bash
     ./gradlew --version    # confirms Gradle + JVM actually in use
     ./gradlew projects     # authoritative module tree — cross-check against step 2
     ./gradlew :app:dependencies --configuration releaseRuntimeClasspath   # optional skim aid
     ```

     (Windows: `.\gradlew.bat …`; substitute the real app module path.) Inspect and quote the
     output in DISCOVERY.md as needed, but do **not** save dependency dumps as files — the
     persistent `docs/factory/audits/deps-<config>.txt` dumps are P3's artifact per the
     artifact-contract matrix in `01-core/CONVENTIONS.md`. If any task fails, record the first
     error verbatim and continue; failure here does not stop discovery.

4. **Manifest reading.** For each module manifest, and primarily the app module:
   - Application ID: AGP ≥ 7 puts it in the build file (`namespace`, `applicationId`); older
     projects use `package=` in the manifest. Cite where you found it.
   - `<application>` attributes: `android:name` (custom Application class), `allowBackup`,
     `usesCleartextTraffic`, `networkSecurityConfig`, `theme`.
   - Enumerate all `<activity>`, `<service>`, `<receiver>`, `<provider>` with `exported` status
     and intent filters (note deep links / `<intent-filter>` with `autoVerify`).
   - All `<uses-permission>` entries, flagging dangerous/runtime permissions and anything unusual
     (`MANAGE_EXTERNAL_STORAGE`, `QUERY_ALL_PACKAGES`, `SYSTEM_ALERT_WINDOW`).
   - `<uses-feature>`, `<queries>`, `<uses-sdk>` overrides, `<meta-data>` (ad network app IDs,
     Firebase, etc.).

5. **Dependency surface skim.** This is a skim, not the full analysis (that is P3 via
   `02-engineering/03-dependency-analysis.md`). From the version catalog and/or `dependencies {}`
   blocks, list every direct dependency with its declared version. Group by concern: AndroidX/UI,
   DI, networking, persistence, async, image loading, ads/monetization, analytics/crash, billing,
   play services, test. Flag immediately-recognizable legacy markers per the red-flags table in
   `02-engineering/01-discovery.md`: `com.android.support:*`, RxJava 1/2, `AsyncTask` usages,
   Butter Knife, EventBus, `kotlin-android-extensions`.

6. **Entry points.** Identify:
   - The `Application` subclass (`android:name` from step 4) — read it; note initialization work
     (DI setup, SDK inits, `StrictMode`).
   - Launcher activity: the `<activity>` with `MAIN`/`LAUNCHER` intent filter — read it; note
     whether it is a single-activity Compose host, a splash, or a classic multi-activity entry.
   - Other externally reachable surfaces: exported components, deep-link activities, widgets,
     `WorkManager` initializers, `androidx.startup` initializers.

7. **UI toolkit detection.** Decide Compose / XML / hybrid with evidence:
   - Grep for `@Composable` in `**/*.kt` — count files and hits.
   - Glob `**/src/main/res/layout/*.xml` — count layout files.
   - Check build files for `buildFeatures { compose = true }` / `viewBinding` / `dataBinding`,
     and the Compose BOM or compiler plugin in dependencies.
   - Verdict: pure Compose (0 layouts), pure XML (0 composables), or hybrid (report the ratio,
     e.g. "41 layouts / 12 composable files — XML-dominant hybrid"). Note Fragment usage count.

8. **CI / flavors / signing detection.**
   - CI: summarize each workflow file found in step 1 — triggers, what it builds/tests/publishes.
     This fills the CI row of PROJECT_STATE.md's Tech Stack Snapshot in P2.
   - Flavors & build types: read `productFlavors {}` and `buildTypes {}` from the app module;
     list every variant axis and resulting variants.
   - Signing: report whether `signingConfigs {}` exists, where credentials are sourced from
     (env vars, `local.properties`, checked-in keystore — flag a checked-in keystore as a finding
     to carry into P3). **Never print passwords, key aliases with secrets, or keystore contents.**

9. **Test presence.** Count files under `**/src/test/**` and `**/src/androidTest/**` per module;
   identify frameworks (JUnit4/5, Robolectric, Espresso, Compose UI test, Turbine, MockK/Mockito)
   from imports or dependencies. Note CI test execution (or its absence) from step 8. Do not run
   tests.

10. **Store presence check.** Determine where the app currently lives:
    - From `applicationId`, check Google Play (`https://play.google.com/store/apps/details?id=<id>`)
      via WebFetch if network is available; also Huawei AppGallery and RuStore per
      `04-aso/stores/`. If network access is unavailable, mark store presence `Unknown — verify`.
    - Look for in-repo evidence: `fastlane/metadata/`, Play listing exports, `versionCode`/
      `versionName` history in git tags (`git tag --list`).

11. **Optional build attempt** (orchestrator only; at most once). Per the P1 build policy in
    `01-core/CONVENTIONS.md`, you may attempt a single debug assembly to establish whether the
    project builds from a clean checkout:

    ```bash
    ./gradlew assembleDebug
    ```

    (Windows: `.\gradlew.bat assembleDebug`.)
    - **Success:** record "builds: yes" with duration and the produced APK path in the Build
      System section.
    - **Failure:** record "builds: no" and the first error verbatim. **Do not fix anything** —
      no SDK-path edits, no version bumps, no `local.properties` creation. The failure is a fact
      for P3 to grade, not a problem for P1 to solve.
    - **Skipped** (no JDK, operator forbade it, obvious multi-minute native build): record
      "build attempt skipped: <reason>". The P1 gate accepts "builds OR build failure documented";
      a documented skip is acceptable only with the operator's knowledge.
    Transient outputs under `build/` and Gradle caches are expected side effects; never commit
    them and never count them as artifacts.

12. **History snapshot (if git available).** `git log --oneline -15`, `git remote -v` (redact
    tokens in URLs), date of last commit, default branch, approximate commit count
    (`git rev-list --count HEAD`). This calibrates how alive the project is.

13. **Assemble DISCOVERY.md.** Merge orchestrator findings and module subagent summaries into the
    output structure below. Re-verify any number you did not personally read from a file. Mark
    every non-cited statement with the literal prefix `Inference:` and every unknown with
    `Unknown — verify`.

## Expected Outputs

One artifact — the **only** artifact of P1 — written into the **target repo**:

`docs/factory/audits/DISCOVERY.md` (create `docs/factory/audits/` if absent) with exactly these
H2 sections, in order:

1. `## Summary` — 5–10 lines: what the app is, size, health one-liner, discovery date (2026-06-10
   format), discovered-by note.
2. `## Repository Layout` — depth-2 tree plus notable top-level files.
3. `## Modules` — table (module, type, namespace, plugins, SDK levels) followed by the per-module
   structured summaries from subagents (or self, if single-threaded).
4. `## Build System` — table: Component | Version | Evidence (`path:line`) for Gradle, AGP, Kotlin,
   JDK, kapt/KSP, version catalog yes/no, repositories — closed by the build-attempt outcome line
   from step 11 (`builds: yes/no/skipped` + evidence).
5. `## Manifest Surface` — applicationId, Application class, components table (name, type,
   exported, intent filters), permissions table with risk notes.
6. `## Dependency Surface` — grouped dependency table with declared versions and legacy flags.
7. `## Entry Points & Navigation` — Application class behavior, launcher flow, exported surfaces.
8. `## UI Toolkit` — verdict with counts and evidence.
9. `## Build Variants, Signing & CI` — variants matrix, signing source, CI summary.
10. `## Tests` — per-module counts and frameworks.
11. `## Store Presence` — per-store status with URLs or `Unknown — verify`.
12. `## Open Questions` — every `Unknown — verify` item collected in one checklist for the operator.

P1 does **not** create `docs/factory/PROJECT_STATE.md` (that is P2's job) and writes no other
files. Do not modify any target source or build file; the only on-disk side effects allowed
beyond DISCOVERY.md are uncommitted Gradle outputs from step 3/11.

## Acceptance Criteria

- [ ] Every module listed in `settings.gradle[.kts]` appears in the Modules section; orphan
      directories and broken includes are explicitly called out (or "none" stated).
- [ ] Gradle, AGP, Kotlin, and JDK versions are each cited with `path:line` from a real file —
      no version appears without evidence.
- [ ] The Build System section records the build-attempt outcome: `builds: yes` (with APK path),
      `builds: no` (with the first error verbatim), or `builds: skipped` (with reason) — satisfying
      the P1 gate "builds OR build failure documented".
- [ ] Application ID, Application class, and launcher activity are identified with file citations.
- [ ] Every `<uses-permission>` in the app manifest appears in the report.
- [ ] UI toolkit verdict includes both the composable count and layout-file count.
- [ ] Every statement that is not directly cited is prefixed `Inference:`; every unknown value
      reads `Unknown — verify` and is mirrored in Open Questions. Zero unmarked speculation.
- [ ] No secrets, keystore passwords, or `local.properties` contents appear anywhere in the report.
- [ ] The report contains zero `{{...}}` tokens and zero TODO/TBD markers.
- [ ] No tracked file in the target repo outside `docs/factory/audits/DISCOVERY.md` was created or
      changed; no target source or build file was modified to make the build attempt pass.

## Documentation Update Rules

**P1 writes only `docs/factory/audits/DISCOVERY.md` — nothing else.**

- `docs/factory/PROJECT_STATE.md` does not exist yet at P1; it is created in P2 by
  `01-core/prompts/02-project-memory.md` from this report. P1 never creates, edits, or
  pre-populates it — not even its Phase Tracker row.
- `docs/factory/CHANGELOG.md` likewise does not exist yet and is not created here; P2's
  onboarding entry records both P1 and P2.
- **Re-discovery** (running this prompt again on a project already onboarded): still write only
  DISCOVERY.md — regenerate it in place, and include in its Summary a short "material diffs from
  previous discovery" list (new modules, version changes). Then tell the operator to run
  `01-core/prompts/02-project-memory.md` in refresh mode; that prompt owns all PROJECT_STATE.md
  and CHANGELOG.md updates. Single-writer discipline keeps project memory uncorrupted.

## Failure Modes & Recovery

1. **No `settings.gradle[.kts]` found.** The path is wrong, or it is not a Gradle project (Bazel,
   Flutter, React Native shell). Search one directory deeper; check for `pubspec.yaml` /
   `package.json` / `WORKSPACE`. Report the actual project type to the operator and stop — the
   factory's pipeline assumes Gradle.
2. **Versions declared indirectly** (convention plugins in `build-logic/`, `buildSrc` constants,
   properties files). Follow the indirection: read the convention plugin source and cite *that*
   file:line. Never report "defined in convention plugin" as a final answer when the plugin source
   is in the repo.
3. **The build attempt fails and you itch to fix it** (missing SDK path, wrong JDK, one-line
   version bump away). Do not. Document the first error verbatim and move on — P1's contract is
   "failure documented, not fixed". A "helpful" fix in P1 corrupts the evidence base P3 grades
   and violates the read-only contract every later phase assumes.
4. **Subagent summaries conflict or come back malformed.** Re-run the offending module yourself,
   single-threaded, citing files directly. Never paste a malformed summary into DISCOVERY.md.
5. **Giant repo blows up context** (hundreds of modules, generated code). Restrict reads: skim the
   version catalog and settings file first, sample representative modules per layer (app, one
   feature, one core lib), and state the sampling strategy in the Summary. Fan out more aggressively
   (still ≤ 8 concurrent) rather than reading everything in one context.
6. **Store lookup blocked** (no network, region lock, app unpublished). Mark each store
   `Unknown — verify` with the exact URL the operator should open manually; do not infer
   published/unpublished from absence of evidence.
7. **Tempted to recommend fixes.** Discovery is descriptive only. Park every "this should change"
   instinct as a neutral observation (e.g., "uses kapt — see P3") so the modernization audit
   (`01-core/prompts/03-modernization-audit.md`) can judge it with full method.
