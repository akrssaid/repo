# CI Setup — Minimal Pipeline for Factory-Managed Projects

A factory-managed app gains CI so that the Tier 2 gate (`02-engineering/07-build-verification.md`)
runs on every push instead of only when an agent remembers, and so regressions the factory just
fixed (kapt, JCenter) cannot silently creep back. This method defines the minimal, copy-pasteable
pipeline and its doctrine. Scales and conventions per `01-core/CONVENTIONS.md`.

## 1. Doctrine

- **When**: CI is *recommended at P4 exit* — after the toolchain is modern (JDK 17, AGP 8.9+,
  Gradle 8.13+), so the pipeline encodes the new baseline, not the legacy one. Earlier is wasted
  motion (the pipeline would churn with every migration); later leaves P6/P8 unguarded.
- **Escalation rule**: adding CI infrastructure to a target repo is a repo-level/process change —
  **operator approval first**, recorded in the `PROJECT_STATE.md` Decision Log (provider, plan
  limits, who pays for minutes). The agent prepares the workflow file on a `factory/p4-ci`
  branch and stops there until approved.
- **Scope**: CI runs Tier 2 always, Tier 3's *build* subset on demand. It never signs, never
  uploads to any store, never touches rollout — those are human-only (`01-core/CLAUDE_MASTER.md`
  escalation triggers; `10-releases/release-workflow.md` division of labor).

## 2. GitHub Actions — the canonical minimal workflow

Complete file; commit as `.github/workflows/ci.yml` in the target repo:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  guards:
    name: Regression guards
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: No kapt reintroduction (factory migrated to KSP)
        run: "! grep -rn --include='*.gradle*' -E 'kapt\\(|id\\(\"kotlin-kapt\"\\)|kotlin\\.kapt' ."
      - name: No JCenter reintroduction
        run: "! grep -rn --include='*.gradle*' 'jcenter()' ."

  build:
    name: Tier 2 (assemble + lint + unit tests)
    needs: guards
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4

      - name: Validate Gradle wrapper
        uses: gradle/actions/wrapper-validation@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Gradle (build cache + config cache)
        uses: gradle/actions/setup-gradle@v4
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}

      - name: Tier 2 gate
        run: ./gradlew lintDebug testDebugUnitTest assembleDebug

      - name: Upload reports on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: reports
          path: |
            **/build/reports/lint-results-*.html
            **/build/reports/tests/
          retention-days: 14
```

Notes: `wrapper-validation` blocks a tampered `gradle-wrapper.jar` (supply-chain guard — keep it
first); `cache-read-only` off main prevents PR branches from poisoning the shared cache;
`timeout-minutes` caps runaway minutes on a hung daemon; lint/test/assemble order surfaces the
cheap failures first while still one Gradle invocation (task graph dedupes configuration). If
the project has product flavors, substitute the default-flavor variants
(`lint<Flavor>Debug test<Flavor>DebugUnitTest assemble<Flavor>Debug`).

## 3. GitLab CI variant

Compact equivalent, `.gitlab-ci.yml`:

```yaml
image: eclipse-temurin:17-jdk

variables:
  GRADLE_USER_HOME: "$CI_PROJECT_DIR/.gradle"
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

cache:
  key: gradle-$CI_COMMIT_REF_SLUG
  paths: [.gradle/caches, .gradle/wrapper]

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

guards:
  stage: .pre
  script:
    - "! grep -rn --include='*.gradle*' -E 'kapt\\(|kotlin\\.kapt' ."
    - "! grep -rn --include='*.gradle*' 'jcenter()' ."

tier2:
  stage: build
  interruptible: true
  script:
    - ./gradlew lintDebug testDebugUnitTest assembleDebug
  artifacts:
    when: on_failure
    paths: ["**/build/reports/"]
    expire_in: 14 days
```

GitLab images need the Android SDK only if instrumented tests are added later; plain
assemble/lint/unit-test downloads SDK components via Gradle's auto-provisioning when
`sdkmanager` licenses are pre-accepted — for simplicity use a community Android image
(pin a digest) or add a `sdkmanager --licenses` bootstrap step if license errors appear.

## 4. Secrets Doctrine — Signing Stays Human

**The keystore never enters CI for factory projects.** Signing is a human-only act per
`01-core/CLAUDE_MASTER.md` (escalation trigger 1: signing keys and store credentials), and the
release workflow's division of labor keeps upload/rollout with the operator. CI therefore builds
**unsigned** release artifacts only (§6); there is nothing secret in the minimal pipeline above —
that is a feature, not a gap.

If the **operator explicitly insists** on CI signing (their decision, logged in the Decision
Log), the standard shape is base64-encoded keystore + passwords as encrypted CI secrets:

```yaml
      # Operator-mandated signing — NOT factory default. Risks the operator accepts:
      # - any maintainer/runner compromise can exfiltrate the upload key
      # - rotate via Play "upload key reset" if leaked; a leaked APP SIGNING key is unrecoverable
      - name: Decode keystore
        run: echo "$KEYSTORE_B64" | base64 -d > upload.jks
        env: { KEYSTORE_B64: ${{ secrets.UPLOAD_KEYSTORE_B64 }} }
      - name: Build signed bundle
        run: ./gradlew bundleRelease
        env:
          KEYSTORE_FILE: ${{ github.workspace }}/upload.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
```

with the matching `signingConfig` reading `System.getenv(...)` and falling back to local
`keystore.properties` (never committed — per `02-engineering/README.md` secrets rule). Use the
**upload key** only, never the app signing key; restrict the secrets to a protected
environment/branch; never echo them; remember `upload.jks` lands on the runner disk.

## 5. Guard Greps (regression tripwires)

Already embedded in §2/§3 as one-line steps. The pattern (`! grep ...` — grep finding nothing
exits 1, negation makes that the pass) is the cheapest possible enforcement of completed
migrations. Extend the same way per project as P4 closes items, e.g.:

```bash
! grep -rn --include='*.gradle*' "compile '"            # ancient 'compile' configuration
! grep -rn --include='*.kt' "kotlinx.android.synthetic" # synthetics removal stays removed
! grep -rn --include='*.kt' "AsyncTask" app/src/main    # if the roadmap eliminated it
```

```powershell
# Local equivalent check (CI runs the bash form on Linux runners):
if (Get-ChildItem -Recurse -Include *.gradle,*.gradle.kts | Select-String 'jcenter\(\)|kapt\(') { throw "guard tripped" }
```

Keep guards in lockstep with the Backlog: a guard exists only for migrations actually completed,
otherwise it blocks unrelated PRs.

## 6. Release-Build Job (Tier 3 subset, unsigned)

Catches R8/shrinking breakage (`02-engineering/09-r8-shrinking.md`) without waiting for a human
Tier 3 run. Manual trigger plus release branches; appended to §2's workflow:

```yaml
  release-build:
    name: Tier 3 subset (bundleRelease, unsigned)
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/heads/factory/p8')
    needs: build
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
      - uses: gradle/actions/wrapper-validation@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '17' }
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew bundleRelease
      - name: Upload AAB + R8 mapping
        uses: actions/upload-artifact@v4
        with:
          name: release-artifacts
          path: |
            app/build/outputs/bundle/release/*.aab
            app/build/outputs/mapping/release/mapping.txt
          retention-days: 30
```

(Add `workflow_dispatch:` under `on:` to enable the manual trigger.) This is a **subset** of
Tier 3: it proves the shrunk build compiles and produces a non-trivial mapping; the on-device
smoke flows, upgrade-path test (`11-data-migration-safety.md` §5), and size gate still run
per `07-build-verification.md` before any release. The unsigned AAB artifact is for inspection
only — the shippable AAB is built and signed by the human per release workflow stage 2. The
archived mapping for a real release comes from that signed build, not from CI.

## 7. Wiring into PROJECT_STATE

The Tech Stack Snapshot in `docs/factory/PROJECT_STATE.md` has a **CI row** (per
`01-core/PROJECT_STATE_TEMPLATE.md`). Fill it when the pipeline lands and keep it current:

```markdown
| CI | GitHub Actions: `.github/workflows/ci.yml` — Tier 2 on PR+main, guards (kapt/jcenter), Tier 3 subset on dispatch | green @ 2026-06-10 |
```

Plus: README badge (optional, operator's call):
`![CI](https://github.com/<owner>/<repo>/actions/workflows/ci.yml/badge.svg)` — and a Decision
Log row for the approval (§1). Every later change to the workflow is a normal factory change:
CHANGELOG entry, and the CI row's status/date refreshed. Prompt 15 treats a red main-branch
pipeline like any red build: a blocker, never shipped around.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| `ci.yml` / `.gitlab-ci.yml` | target repo | Continuous Tier 2 (`07-build-verification.md`); PR hygiene |
| Guard-grep steps | same files | Permanence of P4 migrations; `07-checklists/engineering-modernization.md` |
| Release-build artifacts (unsigned AAB + mapping) | CI artifact storage, 30 d | R8 inspection (`09-r8-shrinking.md`); pre-flight confidence for P8 |
| CI row + Decision Log entry | `PROJECT_STATE.md` | Prompt 15 evidence; factory-version upgrades |
