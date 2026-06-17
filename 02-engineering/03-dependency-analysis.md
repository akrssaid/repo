# Dependency Analysis Method

This method produces a complete, classified picture of everything the target app depends on: the
full resolved graph per configuration, a status verdict for every direct dependency, a replacement
map for everything deprecated or abandoned, transitive conflict detection, and a version-catalog
adoption assessment. It runs in phase P3 under `01-core/prompts/03-modernization-audit.md`, and its
classified table is the primary raw material for ENG- findings and the P4 roadmap
(`01-core/prompts/04-roadmap-generation.md`). Rule: classify **direct** dependencies exhaustively;
inspect transitives only for conflicts and known-bad artifacts.

## Step 1 — Dump the Graph

Dump naming is canonical (`01-core/CONVENTIONS.md`): `docs/factory/audits/deps-<config>.txt`,
where `<config>` is the exact Gradle configuration name.

```bash
./gradlew :app:dependencies --configuration releaseRuntimeClasspath > docs/factory/audits/deps-releaseRuntimeClasspath.txt
./gradlew :app:dependencies --configuration debugRuntimeClasspath  > docs/factory/audits/deps-debugRuntimeClasspath.txt
./gradlew :app:dependencies --configuration releaseCompileClasspath > docs/factory/audits/deps-releaseCompileClasspath.txt
# Per additional module with external deps:
./gradlew :core:data:dependencies --configuration releaseRuntimeClasspath
# Why a given artifact is present (conflict forensics):
./gradlew :app:dependencyInsight --configuration releaseRuntimeClasspath --dependency okhttp
```

On Windows PowerShell, substitute `.\gradlew.bat`; the redirects are identical.

Reading the dump: `->` marks version conflicts resolved upward (`com.squareup.okio:okio:3.6.0 ->
3.9.1`), `(*)` marks repeated subtrees, `(c)` marks catalog/platform constraints. Diff debug vs
release dumps to spot debug-only tooling (LeakCanary, Flipper, Chucker) — these must be
`debugImplementation`; if any appear in the release dump, raise a High ENG- finding.

Direct dependencies come from the build files themselves (`dependencies {}` blocks plus
`libs.versions.toml`), not the dump — the dump shows the *resolved* world, the build files show
*intent*.

## Step 2 — Classify Every Direct Dependency

Determine latest stable per artifact: check the Maven coordinates on `search.maven.org` /
Google Maven (`maven.google.com`), or run the ben-manes plugin if the build allows adding it
temporarily (`./gradlew dependencyUpdates -Drevision=release`). Cross-check Jetpack versions
against `08-knowledge/android/version-matrix.md`. Fill this table — one row per direct dependency,
no exceptions:

| Dependency | Current | Latest stable | Status | Severity | Action |
|---|---|---|---|---|---|

**Status definitions (use exactly these five):**

| Status | Definition | Default severity |
|---|---|---|
| Current | On latest stable, or one minor behind | — |
| Stale | >1 minor or any major behind, but actively maintained | Low (Medium if >2 years behind) |
| Deprecated | Maintainer has declared a successor; still published | Medium |
| Abandoned | No release in 3+ years, archived repo, or jcenter-only | High |
| Security-risk | Known CVE in the used version, or pulls vulnerable transitive | Critical |

For Security-risk: cross-check versions against OSV (`osv.dev`) / GitHub advisories for the exact
coordinates. If the project has CI, recommend adding `org.owasp.dependencycheck` or GitHub
Dependabot as a roadmap item rather than running heavyweight scans during audit.

## Step 3 — Known-Abandoned / Deprecated Checklist

Grep the build files and dumps for each of these. Any hit is an automatic finding at the listed
severity.

| Artifact / signal | Why dead | Severity |
|---|---|---|
| `com.jakewharton:butterknife` | Archived; replaced by ViewBinding/Compose | High |
| `io.reactivex:rxjava` (RxJava **1**, `rx.` packages) | EOL 2018; no fixes | High |
| `com.loopj.android:android-async-http` (AsyncHttpClient) | Abandoned; pre-TLS-modern networking | High |
| `org.jetbrains.anko` | Officially discontinued by JetBrains | High |
| `kotlin-android-extensions` / synthetics imports | Removed in Kotlin 1.8 | High |
| `com.android.support:*` (pre-AndroidX) | Frozen since 2018 | Critical |
| `com.google.android.gms:play-services-ads` very old (<22) | Misses Privacy Sandbox / UMP requirements | High |
| `com.squareup.picasso` on 2.x stale builds | Maintenance-dormant; prefer Coil | Medium |
| `com.github.bumptech.glide` with kapt compiler | Works, but kapt blocks KSP migration; Glide has KSP support — or move to Coil on Compose | Medium |
| jcenter-only artifacts (resolve fails without `jcenter()`) | Repository sunset; supply-chain risk | High |
| `com.facebook.fresco` legacy versions | Heavy; native libs need 16 KB page-size check | Medium |
| `org.apache.httpcomponents` / `useLibrary 'org.apache.http.legacy'` | Removed from Android SDK | High |
| `com.googlecode.*`, `net.sourceforge.*` coordinates | Almost always pre-2015 artifacts | High |
| GSON in new code paths | Maintenance mode; kotlinx.serialization or Moshi preferred (keep if working — Low) | Low |
| `android.enableJetifier=true` | Indicates at least one support-library transitive remains; find it — bash: `./gradlew :app:dependencies \| grep com.android.support`; powershell: `.\gradlew.bat :app:dependencies \| Select-String com.android.support` | Medium |

## Step 4 — Replacement Map

For every Deprecated/Abandoned hit, record the migration target and the guide that owns the work:

| Deprecated | Modern replacement | Migration guide |
|---|---|---|
| Butterknife | ViewBinding (interim) → Compose (target) | `02-engineering/migrations/kotlin-migration.md`; Compose path via `03-design/05-material3-modernization.md` |
| RxJava 1/2/3 | Coroutines + Flow | `02-engineering/migrations/kotlin-migration.md` |
| AsyncTask / AsyncHttpClient | Coroutines + Retrofit/OkHttp | `02-engineering/migrations/kotlin-migration.md` |
| kapt (any processor) | KSP equivalents (Room, Hilt, Glide, Moshi all have KSP) | `02-engineering/migrations/kotlin-migration.md` |
| kotlin-android-extensions | ViewBinding + `@Parcelize` via `kotlin-parcelize` | `02-engineering/migrations/kotlin-migration.md` |
| SharedPreferences | DataStore (Preferences or Proto) | `02-engineering/migrations/android-sdk-migration.md` |
| AsyncTask-era Services / JobScheduler | WorkManager | `02-engineering/05-service-inventory.md` risk table → roadmap |
| Picasso (stale) | Coil 3 (Compose-first) | direct swap; note OkHttp version alignment |
| GSON | kotlinx.serialization (or Moshi) | swap at module boundary; keep DTO tests |
| Support libraries | AndroidX | `02-engineering/migrations/agp-migration.md` precondition |
| Anko layouts/commons | Compose / ktx extensions | `02-engineering/migrations/kotlin-migration.md` |
| ext-block versions / buildSrc constants | `gradle/libs.versions.toml` | `02-engineering/migrations/gradle-migration.md` |

Sequencing note: replacements that unlock other migrations (synthetics, kapt, support-libs,
jcenter) get roadmap priority over like-for-like swaps (Picasso→Coil), regardless of severity
ties. The roadmap prompt sorts by (severity, unblocking-power, effort).

## Step 5 — Transitive Conflict Detection

1. Grep the dumps for `->` lines; group by artifact.
2. For each conflicted artifact run `dependencyInsight` (Step 1) to see who requests what.
3. Verdict per conflict:

| Pattern | Verdict |
|---|---|
| Minor-version bumps within same major (okio 3.6 → 3.9) | Benign; note only |
| Major-version jumps (okhttp 3.x → 4.x forced) | Risky — API breakage in the loser's consumer; test that path; consider explicit constraint |
| Kotlin stdlib mixed versions | Align via catalog `kotlin` version + BOM; mismatches cause obscure compile errors |
| Two artifacts providing same classes (e.g., listenablefuture duplicates, annotations jars) | Add `exclude` or the well-known empty-artifact pin; raises Duplicate class build failures otherwise |
| `force`/`strictly` already present in build files | Document why (ask git blame); forced pins hide rot |

## Step 5b — BOM Alignment Check

Platform BOMs eliminate whole classes of conflicts; verify they are used where available:

| Family | BOM coordinates | Check |
|---|---|---|
| Compose | `androidx.compose:compose-bom` | All `androidx.compose.*` deps versionless under `platform(libs.compose.bom)`; any pinned Compose artifact version is a finding (Low) |
| Firebase | `com.google.firebase:firebase-bom` | Same rule for `com.google.firebase:*` |
| OkHttp | `com.squareup.okhttp3:okhttp-bom` | Use if ≥2 okhttp artifacts present |
| Kotlin | implicit via Kotlin Gradle plugin | `kotlin("bom")` only needed when transitives drag mixed stdlib versions |

Mixed BOM + pinned versions in the same family is the most common self-inflicted conflict source
in acquired apps — normalize to BOM-only during the gradle migration
(`02-engineering/migrations/gradle-migration.md`).

## Step 6 — Version-Catalog Adoption Assessment

| State | Assessment | Roadmap action |
|---|---|---|
| `gradle/libs.versions.toml` exists, all modules use `libs.*` aliases | Adopted | None |
| Catalog exists but modules mix `libs.*` with hardcoded strings | Partial | S effort: sweep hardcoded coords into catalog |
| No catalog; versions in root `ext {}` or `buildSrc` | Not adopted | M effort item in `02-engineering/migrations/gradle-migration.md`; do **before** version bumps so every later migration is a one-line catalog edit |

Measure hardcoded vs catalog usage. The hardcoded pattern must match both DSLs: Kotlin
`implementation("...")` and Groovy's paren-less `implementation "..."` / `implementation '...'`:

```bash
grep -rEc "implementation[ (]['\"]" app/build.gradle*    # hardcoded coordinates (Kotlin + Groovy DSL)
grep -rEc "implementation[ (]libs\." app/build.gradle*   # catalog aliases
```

```powershell
(Select-String -Path app\build.gradle* -Pattern 'implementation[ (][''"]').Count   # hardcoded
(Select-String -Path app\build.gradle* -Pattern 'implementation[ (]libs\.').Count  # catalog
```

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Classified dependency table (Step 2) | `docs/factory/audits/ENGINEERING_AUDIT.md` Dependencies section | ENG- findings (one per non-Current row, Critical/High individually, Medium/Low may batch); P4 roadmap |
| Replacement map instance (Step 4) | ENGINEERING_AUDIT.md | Migration sequencing in `04-roadmap-generation.md`; `07-checklists/engineering-modernization.md` |
| Conflict list + insights (Step 5) | ENGINEERING_AUDIT.md appendix | Risk register; verification focus areas for `02-engineering/07-build-verification.md` |
| Catalog assessment (Step 6) | ENGINEERING_AUDIT.md | First roadmap item ordering (gradle migration) |
| Raw dumps (`deps-<config>.txt`) | `docs/factory/audits/` | Re-diffed after each P4 migration to prove no unintended graph drift |
| Ads/analytics/billing artifacts spotted in the graph | Cross-reference list | `02-engineering/04-sdk-inventory.md` seed list |

After every P4 migration, re-run Step 1 and diff against the stored dumps; any new `->` conflict or
new transitive major version is investigated before the migration's verification gate
(`02-engineering/07-build-verification.md`) is considered passed.

Any dependency **addition, removal, or major-version change** involving an SDK that collects data
(ads/analytics/crash/attribution) also triggers the Data-safety re-check per
`04-aso/workflows/data-safety-listing.md` — the declared form must track the shipped graph.
