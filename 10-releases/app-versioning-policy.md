# App Versioning Policy — Target Apps

This is the factory standard for versioning TARGET Android apps: what `versionName` means, how
`versionCode` is computed, how multiple stores share one version, and the procedure for bumping.
Versions are locked in stage 1 of `10-releases/release-workflow.md` (pre-flight), before any
build exists. The consistency checklist at the end is enforced by
`01-core/prompts/16-release-preparation.md`.

## versionName — semver semantics for apps

`versionName` is the user-visible string. It follows `MAJOR.MINOR.PATCH`, interpreted for
consumer apps (not libraries — apps have no API to break, so "breaking change" is redefined):

| Component | Bump when | Examples |
|---|---|---|
| MAJOR | A redesign or major-feature era: full UI overhaul (P6 redesign ships), business-model change, rebrand, rewrite. Users should *notice* a new app. | 1.x → 2.0.0 after the Material 3 redesign |
| MINOR | Feature releases: new user-facing capability, new screen, new format supported. The normal release cadence. | 2.3.0 adds cloud backup |
| PATCH | Fix-only: crash fixes, ANR fixes, translation corrections, dependency security bumps. Zero new user-facing behavior. | 2.3.1 fixes the import crash |

Rules:

1. Decide the bump from the change set, not ambition. If a "feature release" slipped to fixes
   only, ship it as PATCH.
2. Never skip numbers and never go backwards. 2.3.0 → 2.4.0 or 2.3.1 or 3.0.0; nothing else.
3. Pre-release suffixes (`-beta1`) are allowed on closed-testing builds only; production
   versionNames are always bare `X.Y.Z` (plus the store-flavor suffix below).
4. MAJOR = 0 is not used. Acquired/legacy apps enter the factory at whatever MAJOR they had;
   the first factory release usually bumps MINOR (modernization) or MAJOR (full redesign).

## versionCode — the factory standard scheme

`versionCode` is the integer the stores actually compare. It must be strictly monotonic, never
reused, and ≤ 2,100,000,000 (Play's hard ceiling).

**Factory standard: semver-derived `M NN PP BB`:**

```
versionCode = MAJOR × 1,000,000  +  MINOR × 10,000  +  PATCH × 100  +  BUILD
```

- `BUILD` (00–99) counts release-build attempts of the same `X.Y.Z` — it absorbs rejected or
  discarded uploads (see recovery section). First attempt is `01`, never `00`, so a versionCode
  always changes when anything changes.
- Capacity: MINOR and PATCH up to 99, MAJOR up to 2099. No realistic app exhausts this.

**Worked examples:**

| versionName | BUILD | versionCode | Note |
|---|---|---|---|
| 2.3.0 | 1st attempt | 2,030,001 | Normal release |
| 2.3.0 | 2nd attempt | 2,030,002 | First upload rejected; same versionName, new code |
| 2.3.1 | 1st attempt | 2,030,101 | Hotfix |
| 2.10.0 | 1st attempt | 2,100,001 | Two-digit MINOR — no collision (see below) |
| 3.0.0 | 1st attempt | 3,000,001 | Redesign era |

**Why not naive schemes — collision risks:**

| Naive scheme | Failure mode |
|---|---|
| `M*100 + N*10 + P` (one digit per part) | A part exceeding 9 overflows its field and collides: 1.23.0 and 12.3.0 cannot both be encoded, and `2.1.0` (210) sorts below `1.21.0` once widths overflow. Any scheme without fixed-width fields breaks the moment a part exceeds 9. |
| "Just increment by 1 each release" | Monotonic but meaningless: cannot recover which source version a crash report's versionCode maps to without a lookup table; hotfix branches and main fight over the next integer. |
| Date-based `YYMMDDNN` | Monotonic and roomy (26061001 for 2026-06-10 build 01), and a legitimate alternative — but it encodes *when*, not *what*. Two same-day builds of different versionNames are indistinguishable by code; and you cannot later insert a hotfix of an old MAJOR (its date-code would exceed newer releases, force-upgrading everyone). The factory rejects it for multi-branch hotfix safety. |
| Reusing a code after a rejected upload | Play permanently consumes every uploaded versionCode, even from discarded drafts. The re-upload is refused. This is what `BB` exists for. |

If an inherited app already uses a larger scheme (e.g. old code 305000000), continue monotonic:
adopt `M NN PP BB` *plus* a fixed legacy offset (e.g. +300,000,000) documented in
`docs/factory/PROJECT_STATE.md`, and never remove the offset.

## Multi-store strategy

**Standard: same versionCode everywhere.** One build lineage, one integer, every store (Play,
AppGallery, RuStore). Per-store differences are expressed as product flavors whose `versionName`
gains a suffix, never a different code:

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        versionCode = AppVersion.CODE        // 2_030_001 — identical in every flavor
        versionName = AppVersion.NAME        // "2.3.0"
    }
    flavorDimensions += "store"
    productFlavors {
        create("play")    { dimension = "store" }
        create("gallery") { dimension = "store"; versionNameSuffix = "-hms" }
        create("rustore") { dimension = "store"; versionNameSuffix = "-ru" }
    }
}
```

So RuStore shows `2.3.0-ru`, versionCode 2,030,001 — same as Play.

**Why not per-store offsets** (e.g. Play +0, AppGallery +1, RuStore +2): offsets bite, every
time —

1. Crash reports arrive keyed by versionCode; with offsets, one source build has three
   identities and triage must de-offset by hand.
2. The "never reuse" bookkeeping triples; a rejected upload on one store desynchronizes the
   offsets forever.
3. Users migrating between store ecosystems (sideload → store install) hit downgrade errors
   when the offset order opposes the install order.
4. The `M NN PP BB` mapping back to source versions breaks for two of three stores.

The only acceptable per-store code divergence is the `BB` field when a single store rejects an
upload (see recovery) — and then the *next* release re-converges all stores on one code.

## Track / build-type mapping

| Track (Play) | Build type / flavor | versionCode | versionName |
|---|---|---|---|
| Internal testing | `release` (signed) | The real release code, e.g. 2,030,001 | `2.3.0` |
| Closed testing | Same artifact promoted | Same — promotion, not rebuild | `2.3.0` (or `2.3.0-beta1` if built separately) |
| Production | Same artifact promoted | Same | `2.3.0` |
| Local debug | `debug` | Irrelevant (never uploaded) | `2.3.0-debug` via suffix |

Principle: **one artifact climbs the tracks**. Internal → closed → production is the same AAB,
the same versionCode (per `10-releases/release-workflow.md` stage 3). A rebuild for any reason
is a new `BB` and restarts at internal.

## Consistency checklist — where the version must match

Enforced at prompt 16 and re-checked in pre-flight. One versionName + versionCode must be
identical across the **five canonical locations**:

- [ ] **Gradle source of truth** — `AppVersion` constants or version catalog (see below); this is
      the only place the pair is authored.
- [ ] **Git tag** — `release/vX.Y.Z` (per `01-core/CONVENTIONS.md`), annotated with the versionCode.
- [ ] **`docs/factory/PROJECT_STATE.md`** — the **Release History** row (versionName/versionCode,
      date, tag, per-store status) **and** the **Artifact Index** row for the release report (D6).
- [ ] **`docs/factory/CHANGELOG.md`** — entry heading.
- [ ] **Release report** — the `09-templates/release-report.md` instance at close-out
      (`RELEASE_REPORT_v<versionName>.md`).

Two downstream surfaces must also agree but are not authoring locations: the release notes /
store-listing "what's new" headers, and the store consoles after upload (Play, AppGallery, RuStore
show the expected pair). Any mismatch found post-upload is recorded as an incident in the release
report and fixed at the Gradle source of truth before the next release.

## Version bump procedure

The version lives in exactly one place in the build. Two sanctioned patterns:

**Pattern A — version catalog (`gradle/libs.versions.toml`), factory default:**

```toml
[versions]
appVersionName = "2.3.0"
appVersionCode = "2030001"
```

```kotlin
// app/build.gradle.kts
defaultConfig {
    versionName = libs.versions.appVersionName.get()
    versionCode = libs.versions.appVersionCode.get().toInt()
}
```

**Pattern B — buildSrc constants (for projects already carrying buildSrc):**

```kotlin
// buildSrc/src/main/kotlin/AppVersion.kt
object AppVersion {
    const val MAJOR = 2; const val MINOR = 3; const val PATCH = 0; const val BUILD = 1
    const val CODE = MAJOR * 1_000_000 + MINOR * 10_000 + PATCH * 100 + BUILD
    const val NAME = "$MAJOR.$MINOR.$PATCH"
}
```

Pattern B computes CODE from NAME, making drift impossible — prefer it when buildSrc exists.

**Bump steps (stage 1 of the release workflow):**

1. Decide the semver bump from the change set (table above).
2. Edit the single source of truth; reset `BUILD` to 1 for a new `X.Y.Z`.
3. Verify against the **merged manifest** — the only place the build's final `versionCode` is
   resolved. Generate it with the manifest-processing task (never `:app:dependencies`, which
   resolves dependencies, not the manifest), then read it back:

   ```bash
   ./gradlew :app:processDebugMainManifest
   grep versionCode "$(find app/build/intermediates -name AndroidManifest.xml -path '*merged*' | head -1)"
   ```

   ```powershell
   .\gradlew :app:processDebugMainManifest
   Select-String "versionCode" (Get-ChildItem app\build\intermediates -Recurse -Filter AndroidManifest.xml | Where-Object FullName -match 'merged' | Select-Object -First 1).FullName
   ```

   The merged-manifest output path differs by AGP version and is stated once in
   `02-engineering/06-manifest-audit.md` **Step 0** (both AGP-version variants); use that path
   rather than hardcoding it. The value read back must equal the computed code.
4. Commit the bump as its own commit: `chore(release): bump to 2.3.0 (2030001)`.
5. Update `docs/factory/PROJECT_STATE.md` and the changelog heading; tag per the workflow.

## Never reuse a versionCode — and recovery after a rejected upload

**The rule:** a versionCode, once uploaded to *any* store — even to a draft that was discarded,
even if review rejected it — is dead. Play enforces this server-side; the factory enforces it
universally so all stores stay converged.

**Recovery procedure when a build is rejected post-upload** (review rejection, signing mismatch,
forgotten debug flag — any reason the uploaded artifact cannot ship):

1. Do not change the versionName — the release is still `2.3.0` to users and to the changelog.
2. Fix the cause; increment `BUILD`: 2,030,001 → 2,030,002. Commit, re-tag is NOT needed (the
   tag `release/v2.3.0` is moved only if the *source* changed: delete and re-create the
   annotated tag on the fix commit, noting the new code in the tag message).
3. Rebuild and restart the workflow at stage 2 (Build & Sign); the new artifact re-enters
   internal testing — never upload an untested rebuild straight to production.
4. Record the burned code in `docs/factory/PROJECT_STATE.md` release history ("2030001 —
   burned, upload rejected: <reason>") so future audits don't search for a phantom release.
5. If `BUILD` would exceed 99 (pathological), bump PATCH instead and document why.
