# Android Version Matrix

The single reference for Android API levels, the behavior changes that matter when modernizing an
existing app, minSdk reach tradeoffs, Google Play target-API deadline mechanics, and toolchain
compatibility. Consult this during P1 (Discovery), P3 (Engineering Audit), and P4 (Modernization)
whenever an API-level decision is made. Behavior-change rows are stable history; everything marked
volatile drifts — **verify against current official docs** before acting on it.

**last reviewed: 2026-06-10**

---

## API Level Matrix (21 → 36)

Behavior changes listed are those that bite during modernization — i.e. they activate when an app
**targets** that level (T) or apply to **all apps on that OS version** regardless of target (A).

| API | Version | Codename | Year | Notable behavior changes relevant to modernization |
|---|---|---|---|---|
| 21 | 5.0 | Lollipop | 2014 | ART becomes the only runtime; Material Design introduced; `JobScheduler` added; WebView becomes updatable via Play. Practical floor for legacy support. |
| 22 | 5.1 | Lollipop MR1 | 2015 | Multiple SIM support; minor. Almost never a meaningful minSdk boundary. |
| 23 | 6.0 | Marshmallow | 2015 | **Runtime permissions** (T): dangerous permissions must be requested at use time. Doze and App Standby (A) start deferring background work and alarms. Apache HTTP client removed from default classpath. |
| 24 | 7.0 | Nougat | 2016 | `file://` URIs across apps throw `FileUriExposedException` (T) — `FileProvider` required. Multi-window. Doze extended ("Doze on the go"). Lowest installable target for many modern Jetpack libs. |
| 25 | 7.1 | Nougat MR1 | 2016 | App shortcuts. Minor as a boundary. |
| 26 | 8.0 | Oreo | 2017 | **Background execution limits** (T): background services killed shortly after app leaves foreground — `startService` from background throws; WorkManager/FGS territory. **Notification channels required** (T) — notifications without a channel are dropped. Implicit broadcast receivers largely banned from the manifest (T). Adaptive icons introduced. |
| 27 | 8.1 | Oreo MR1 | 2017 | Minor; notification badge behaviors. |
| 28 | 9 | Pie | 2018 | Non-SDK (hidden) API restrictions begin (A, graylist by target). **Cleartext HTTP blocked by default** (T) — needs `networkSecurityConfig` to opt back in per-domain. Display cutout APIs. Apache HTTP needs explicit `uses-library`. |
| 29 | 10 | Q | 2019 | **Scoped storage** introduced (T): raw external-storage paths stop working; `requestLegacyExternalStorage` opt-out exists at 29 only. Background activity starts restricted (A). Gesture navigation arrives — edge swipes conflict with drawer/edge UI. Location "while in use" option. |
| 30 | 11 | R | 2020 | **Scoped storage enforced** (T) — legacy flag ignored; `MANAGE_EXTERNAL_STORAGE` requires Play policy justification. **Package visibility** (T): `queryIntentActivities`/`getInstalledPackages` return filtered results unless `<queries>` declared. One-time permissions; auto-reset of unused app permissions (A). Background location must be requested separately. |
| 31 | 12 | S | 2021 | **`android:exported` must be explicit** (T) for every component with an intent filter — install fails otherwise. PendingIntent mutability (`FLAG_IMMUTABLE`/`FLAG_MUTABLE`) mandatory (T). `SCHEDULE_EXACT_ALARM` permission for exact alarms (T). SplashScreen API replaces hand-rolled splash activities (A — system splash always shown). Dynamic color / Material You (A). Approximate-location user option. |
| 32 | 12L | S V2 | 2022 | Large-screen/foldable focus; taskbar. No major target-gated breakage; treat as 31 for audits. |
| 33 | 13 | Tiramisu | 2022 | **`POST_NOTIFICATIONS` runtime permission** (T) — notifications silently dropped until granted. Granular media permissions (`READ_MEDIA_IMAGES/VIDEO/AUDIO`) replace `READ_EXTERNAL_STORAGE` (T). Photo Picker available (A). Per-app language preferences. Predictive-back opt-in begins. |
| 34 | 14 | UpsideDownCake | 2023 | **Foreground service types mandatory** (T): every FGS must declare a type in manifest + corresponding permission, and the use case must match Play policy. Implicit intents cannot resolve to non-exported components (T). Runtime-registered broadcast receivers must flag exported state (T). Partial photo/video access ("Select photos"). OS refuses to install apps targeting < 23 (A). |
| 35 | 15 | VanillaIceCream | 2024 | **Edge-to-edge enforced by default** (T): status/nav bars draw over the app; inset handling mandatory — see `08-knowledge/android/common-pitfalls.md`. 16 KB memory-page-size compatibility becomes a real requirement for native libs. Predictive back further rolled out. `TYPE_DATA_SYNC` FGS timeout. |
| 36 | 16 | Baklava | 2025 | **Adaptive apps** (T): orientation/resizability/aspect-ratio restrictions ignored on large screens (≥600dp) — locked-portrait utility apps must handle landscape/resizable layouts. Predictive back animations on by default (T). Edge-to-edge opt-outs removed (T). 16 KB page-size alignment required for native code on supported devices. Material 3 Expressive components arrive in Compose M3 alphas. |

The factory tech baseline (restated authoritatively in `01-core/CLAUDE_MASTER.md`):
**compileSdk = targetSdk = 36, minSdk 26 recommended (24 acceptable for legacy reach)**.

### The three SDK values — what each actually gates

| Value | Means | Audit rule |
|---|---|---|
| `compileSdk` | Which API surface the code compiles against. Raising it alone changes nothing at runtime | Always raise first, freely — it only unlocks newer APIs and lint checks |
| `targetSdk` | Which behavior changes (the **T** rows above) the OS applies to the app | Raise one level at a time through the matrix, fixing each level's T-row items before the next; this is the actual migration work in `02-engineering/migrations/android-sdk-migration.md` |
| `minSdk` | Oldest installable OS; gates which APIs need `Build.VERSION` guards | Business decision — use the reach table below; raising it deletes guard code, lowering it is almost never justified |

When auditing, read all three from the module's `build.gradle.kts` (or the version catalog) and
record the gap between `targetSdk` and the current baseline as the headline modernization-debt
number: each level of gap maps to a concrete T-row work package above.

---

## Device Distribution & minSdk Tradeoff

Distribution numbers are volatile — **verify against current official docs**. Where to check:

1. Android Studio: New Project wizard → "Help me choose" API-level distribution chart.
2. Google Play Console → an existing app → Reach and devices (best source: *your* category's real users).
3. Community mirrors of the platform dashboard (e.g. apilevels.com) for quick reference.

Approximate global reach as of mid-2026 (treat as ±2 points, varies strongly by geography — older
devices concentrate in emerging markets where many utility apps over-index):

| minSdk | Approx. reach | You give up | You gain |
|---|---|---|---|
| 24 (7.0) | ~99% | Almost nothing in reach | Must support pre-channel notifications, pre-adaptive icons; some Jetpack libs now floor at 23–24 anyway |
| 26 (8.0) | ~97–98% | ~1–2% of very old devices | Notification channels and adaptive icons guaranteed; no legacy background-service code paths; **factory recommendation** |
| 29 (10) | ~92–95% | A real slice in emerging markets | Scoped storage universal — no legacy storage branch; gesture nav assumed; simpler permission matrix |

Decision rule: keep the target app's existing minSdk unless it is < 24 (raise to 24 minimum) or
the audit shows < 1% of the app's actual installs below the proposed floor (then raise to 26).
Record the decision and evidence in `docs/factory/audits/ENGINEERING_AUDIT.md`.

---

## Google Play Target-API Deadline Mechanics (the August cycle)

Volatile — **verify against current official docs** (Play Console policy: "Target API level
requirements") before every release cycle, per `08-knowledge/README.md` freshness doctrine.

How the cycle works:

1. A new Android major version finalizes its API (typically Q2–Q3).
2. The following **August 31**, Google Play requires **new apps and all app updates** to target an
   API level within **one year** of the latest major release. As of 2026-06-10: deadline
   2026-08-31 requires targetSdk 35+ for updates; targeting 36 now skips next year's scramble.
3. **Existing apps** that stop updating may keep their listing, but apps targeting more than two
   years behind become **invisible to new users** on devices running newer Android versions.
4. A one-time **extension to November 1** can be requested in Play Console (Policy → App content)
   per app per cycle. Plan around August anyway; the extension is a parachute, not a strategy.
5. Wear OS / TV / Automotive have their own (laxer) schedules — irrelevant to phone utility apps.

Factory consequence: every target app that passes through P4 (Modernization) exits at
targetSdk = current baseline (36), making the August deadline a non-event. Apps in the portfolio
*not* yet modernized must be triaged each June against the upcoming deadline.

---

## AGP ↔ Gradle ↔ Kotlin ↔ JDK Compatibility

**This is the canonical toolchain version table for the factory.** The two migration guides
(`02-engineering/migrations/gradle-migration.md` and `.../agp-migration.md`) and any artifact that
pins toolchain versions cite this table rather than restating numbers.

Volatile — **verify against current official docs** (AGP release notes "compatibility" table,
Kotlin Gradle-plugin compatibility page) before pinning. Minimum required versions:

| AGP | Min Gradle | JDK (to run Gradle) | Notes |
|---|---|---|---|
| 8.0–8.1 | 8.0–8.2 | 17 | First JDK-17-mandatory line; namespace in build file required |
| 8.2–8.3 | 8.2–8.4 | 17 | |
| 8.4–8.6 | 8.6–8.7 | 17 | |
| 8.7–8.8 | 8.9–8.10 | 17 | |
| **8.9.1** (min) – 8.10 | 8.11+ | 17 | **Minimum AGP for `compileSdk 36`** — the factory baseline floor; older AGP + compileSdk 36 = manifest-merger/resource failures (per D11 / `01-core/CONVENTIONS.md`) |
| 9.x | 9.x | 17+ | Built-in Kotlin support changes; verify before adopting |

**The load-bearing rule:** `compileSdk 36` requires **AGP 8.9.1 or newer**. This is the single
number both migration guides and every toolchain-pinning artifact reference; do not restate it
elsewhere — cite this table (and `01-core/CONVENTIONS.md`).

Rules of thumb:

- Upgrade order in P4: **Gradle wrapper → AGP → Kotlin → libraries** (one commit each, build
  green between steps — see `02-engineering/migrations/gradle-migration.md` and
  `02-engineering/migrations/agp-migration.md`).
- JDK 17 is the toolchain baseline; pin it with Gradle toolchains
  (`java.toolchain.languageVersion = 17`) so local JDK drift cannot break builds.
- Kotlin 2.1+ (K2 compiler) is the factory floor; Kotlin and AGP are largely decoupled in the
  8.x/2.x era but check the Kotlin release notes for the supported AGP range.
- compileSdk 36 requires **AGP 8.9.1+** (the canonical floor in the table above); an old AGP with a
  new compileSdk produces warnings first and obscure resource/manifest-merger failures later.

```powershell
# Verify the active toolchain of a target app quickly
.\gradlew --version                                  # Gradle + JVM in use
Select-String -Path gradle\libs.versions.toml -Pattern 'agp|kotlin'   # declared versions
```

---

## Compose BOM ↔ Kotlin Compatibility

Since **Kotlin 2.0**, the Compose compiler ships **inside the Kotlin distribution** via the
`org.jetbrains.kotlin.plugin.compose` Gradle plugin — its version must equal the Kotlin version,
and the old `composeOptions { kotlinCompilerExtensionVersion }` block must be deleted. The Compose
**BOM** (e.g. `androidx.compose:compose-bom:2025.x`) versions only the *libraries* and is no
longer coupled to the Kotlin version.

Pre-2.0 legacy apps instead pin a standalone Compose compiler whose version maps 1:1 to a specific
Kotlin version — the single most common build break when bumping Kotlin on a legacy codebase.
Migration: bump Kotlin to 2.1+, apply the compose compiler Gradle plugin, delete `composeOptions`.
Exact BOM-to-library mappings are volatile — **verify against current official docs**
("Compose to Kotlin Compatibility Map" page).

Cross-references: `02-engineering/migrations/android-sdk-migration.md` (applying target bumps),
`02-engineering/migrations/kotlin-migration.md` (K2 adoption),
`08-knowledge/android/modern-stack.md` (what to build with once the toolchain is current).
