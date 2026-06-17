# Testing Strategy — The Migration Safety Net

Legacy apps arrive with little or no automated testing, and P4 then rewrites their riskiest seams
(annotation processing, persistence, SDK behavior, shrinking). This method defines the **minimum
safety net that must exist before any migration starts** and what "tested" means for each
migration type. It is the prerequisite the roadmap encodes as **WS-0** (`01-core/prompts/
04-roadmap-generation.md`): the safety-net workstream is sequenced before every other workstream
and no migration guide in `02-engineering/migrations/*` may begin until WS-0's gate is green.
Prompts 09, 15, and 16 consume the smoke-flow list authored here; the degraded-gate rules below
replace the "Tier 2 degrades to assemble + lint" note in `02-engineering/07-build-verification.md`
with a real procedure. Scales and IDs per `01-core/CONVENTIONS.md`.

## 1. Characterization Tests Before P4

A characterization test (golden-output test) records what the code **does today** — not what it
should do — so that a migration which silently changes behavior fails a test instead of a user.
Write them only around **seams the roadmap is about to change**; do not attempt coverage.

**Procedure:**

1. Read the Backlog in `docs/factory/PROJECT_STATE.md` and the workstream table from prompt 04.
   List every migration that will run in P4.
2. For each migration, identify its seams from the table in §2 below. Locate the concrete classes
   with targeted greps:

```bash
# Serialization boundaries (kapt→KSP, R8, library bumps all threaten these)
grep -rn --include="*.kt" -E "@SerializedName|@Json\(|@Serializable|TypeToken" app/src
# DAO queries (Room version bumps, KSP migration)
grep -rln --include="*.kt" -E "@Dao|@Query" app/src
# Formatters & parsers (locale/SDK behavior changes)
grep -rn --include="*.kt" -E "SimpleDateFormat|DateTimeFormatter|NumberFormat|String\.format" app/src
```

```powershell
Get-ChildItem app\src -Recurse -Filter *.kt | Select-String -Pattern '@SerializedName|@Json\(|@Serializable|TypeToken'
Get-ChildItem app\src -Recurse -Filter *.kt | Select-String -Pattern '@Dao|@Query' | Select-Object -ExpandProperty Path -Unique
Get-ChildItem app\src -Recurse -Filter *.kt | Select-String -Pattern 'SimpleDateFormat|DateTimeFormatter|NumberFormat|String\.format'
```

3. For each seam, write one test that feeds a **realistic fixed input** and asserts the **exact
   current output** (run once, paste the real output into the assertion — even if it looks wrong;
   wrong-but-shipped behavior is the contract users depend on).
4. Place tests in `app/src/test/.../characterization/`; suffix class names with
   `CharacterizationTest` so they are greppable and removable later.
5. Run `./gradlew testDebugUnitTest` — all green is the WS-0 exit gate, recorded in
   `docs/factory/CHANGELOG.md`.

**Concrete example — pinning Gson output before a kapt→KSP + R8 change:**

```kotlin
// app/src/test/java/.../characterization/InvoiceSerializationCharacterizationTest.kt
class InvoiceSerializationCharacterizationTest {

    private val gson = ApiModule.provideGson()   // the REAL production instance, not a fresh Gson()

    @Test
    fun `invoice json shape is stable`() {
        val invoice = Invoice(
            id = 42L,
            amountCents = 1999,
            currency = "EUR",
            issuedAt = Date(1735689600000L),     // fixed epoch — never Date()
            customerNote = null,
        )
        // Golden string captured from current production code on 2026-06-10. If a migration
        // breaks this test, the WIRE FORMAT changed — old persisted/queued JSON will not parse.
        assertEquals(
            """{"id":42,"amount_cents":1999,"currency":"EUR","issued_at":"2025-01-01T00:00:00Z"}""",
            gson.toJson(invoice),
        )
    }

    @Test
    fun `legacy json with unknown fields still parses`() {
        val legacy = """{"id":7,"amount_cents":500,"currency":"USD","issued_at":"2025-01-01T00:00:00Z","v":2}"""
        val parsed = gson.fromJson(legacy, Invoice::class.java)
        assertEquals(7L, parsed.id)
        assertEquals(null, parsed.customerNote)  // pins current null-handling behavior
    }
}
```

The same pattern applies to a DAO (insert fixtures via in-memory Room, assert exact query
results), a date formatter (fixed `Locale` + fixed millis → exact string), or a parser (real
captured payload file in `src/test/resources/` → exact object graph).

## 2. Minimum Safety Net per Migration Type

The migration guides cite this table; the listed tests must be **green before** the migration
starts and **re-run after** it as part of the mandated verification tier.

| Migration | Must be tested BEFORE | Must be verified AFTER | Tier (`07-build-verification.md`) |
|---|---|---|---|
| kapt→KSP (`migrations/kotlin-migration.md`) | Characterization on every annotation-processed output consumed at runtime: one serialization round-trip per Gson/Moshi model family, one query per Room DAO, Hilt graph builds (app launches) | Same tests green; generated-code diff sanity (`build/generated/ksp/` exists, `build/generated/source/kapt/` gone); app launches (DI graph) | 2 + run app |
| SharedPreferences→DataStore | Characterization that reads every migrated key from a populated SharedPreferences file and asserts values/defaults | Migration test per `02-engineering/11-data-migration-safety.md` §4; upgrade-path install test (same doc §5) | 3 |
| targetSdk hop (`migrations/android-sdk-migration.md`) | Smoke-flow list authored (§5); characterization on anything touching permissions, notifications, alarms, foreground services, storage paths | Full smoke flows on an emulator running the NEW API level *and* minSdk level; behavior-change checklist from the migration guide | 3 |
| R8 enablement (`02-engineering/09-r8-shrinking.md`) | Reflection inventory complete (that doc §3); characterization on every reflective serialization seam | All smoke flows on the **obfuscated release build**; `mapping.txt` spot-checks; zero `ClassNotFoundException` in logcat | 3 |
| Compose interop (View↔Compose, P6 batches) | Screenshot baseline exists (`assets/screenshots/baseline/`); characterization on ViewModel outputs feeding migrated screens | Compose UI test per migrated screen (§3) asserting key nodes; smoke flow through the screen; rotation survives | 2 per batch, 3 at batch-set end |

## 3. Framework Selection

| Framework | Use for | Do NOT use for |
|---|---|---|
| JUnit 4 | Default in legacy projects — keep it; characterization tests run fine on it | Don't block WS-0 on a JUnit 5 migration |
| JUnit 5 | New greenfield test modules where the project already uses it (or `junit-vintage` is wired) | Instrumented tests (AndroidJUnitRunner is JUnit 4) |
| Robolectric | JVM-speed tests needing `Context`, resources, SharedPreferences, Room (in-memory) — the workhorse for characterization of Android-touching seams | Anything rendering-accurate, WebView, real IPC, R8 verification (it tests un-shrunk classes) |
| Instrumented (`androidTest`) | `MigrationTestHelper` (Room), DataStore migration proof, anything where real-framework behavior is the question | Pure logic — too slow as the default |
| MockK | Mocking Kotlin collaborators (final classes, objects, coroutines) at injection boundaries | Mocking types you own and can instantiate; never mock the class under characterization |
| Turbine | Asserting `Flow` emissions in ViewModel/repository tests (`flow.test { awaitItem() }`) | One-shot suspend functions — plain `runTest` suffices |
| Compose UI test (`createAndroidComposeRule`) | Per-screen assertions after Compose interop work: node exists, click navigates, state restores | XML-only screens |
| Espresso | Legacy XML screens that already have Espresso tests — keep them green | New tests on Compose screens (use Compose test APIs; Espresso only for hybrid View interop) |

**Selection rules:** (1) match what the project already has before adding anything; (2) every
added test dependency is a Backlog-visible change — record it in `docs/factory/CHANGELOG.md`;
(3) prefer Robolectric for characterization speed, instrumented only where this table or
`11-data-migration-safety.md` mandates it; (4) no UI-test framework work in WS-0 beyond what §2
requires — depth comes later as recorded test debt (§6).

## 4. Degraded Gate — What Tier 2 Means With Zero Tests

When the architecture-mapping testability score (`02-engineering/02-architecture-mapping.md`)
records **no usable unit tests**, Tier 2 does *not* silently degrade to assemble + lint. The
degraded gate is:

1. **Characterization minimum**: the §2 BEFORE-column tests for every migration in the current
   workstream exist and pass. Absolute floor even for a single-seam change: one characterization
   test on the seam being changed.
2. **Smoke flows**: all P0 flows (§5) executed manually on a debug build after the change, results
   recorded in the CHANGELOG entry.
3. The CHANGELOG entry is marked `Tier 2 (degraded: characterization + P0 smoke)` — never plain
   "Tier 2", so prompt 15 can see exactly what was and wasn't verified.

A degraded gate is a temporary state: every phase that runs under it MUST append test-debt rows
to the Backlog (§6). Running two consecutive phases degraded without Backlog rows is a process
violation prompt 15 flags as FAIL.

## 5. Smoke-Flow Authoring (the list prompts 15/16 execute)

Derive the flow list from `docs/factory/audits/SCREEN_INVENTORY.md` (produced by prompt 05 from
`09-templates/screen-inventory.md`) — its per-screen **Priority/Traffic** column (H/M/L) and Flow
Map section:

- **P0 flows** — every flow whose screens are all Priority H, plus cold start → first screen
  render, plus any flow containing a monetization surface (ads load, purchase sheet opens).
  These run at every Tier 3 pass and every degraded Tier 2 (§4).
- **P1 flows** — flows touching Priority M screens: one representative flow per feature area,
  plus rotation on the main screen, background/resume, one deep link, and the upgrade-path
  install test from `02-engineering/11-data-migration-safety.md` §5.
  These run at Tier 3 only.
- Priority L screens get a single "opens without crash" sweep during P8, not a flow each.

Write the list as a table in `docs/factory/PROJECT_STATE.md` under Backlog (one pinned block) or
directly into the verification report when first executed:

```markdown
| Flow ID | Pri | Screens (SCR-IDs) | Steps | Expected |
|---|---|---|---|---|
| FLOW-P0-1 | P0 | SCR-001 | Cold start from launcher | Main screen interactive < 5 s, no FATAL |
| FLOW-P0-2 | P0 | SCR-001→SCR-003→SCR-007 | Open doc → edit → save | Saved doc reopens with edits |
```

If `SCREEN_INVENTORY.md` does not exist yet (P4 running before P5 — the common order), derive a
provisional P0 list from the launcher activity plus the app playbook's core flow
(`06-playbooks/*`), and mark it `provisional` so prompt 05 replaces it.

## 6. Test-Debt Recording

Every gap this method consciously leaves (screens without UI tests, seams characterized but not
unit-tested properly, degraded-gate phases) becomes a Backlog row in
`docs/factory/PROJECT_STATE.md`:

```markdown
| Add Room DAO unit tests beyond characterization set | Medium | M | P4 | Not Started |
| Compose UI tests for SCR-004/005 (migrated in P6 batch 2) | Medium | S | P6 | Not Started |
```

Severity Medium by default; High if the untested seam guards payments, auth, or user data.
Source phase = the phase that created or discovered the gap. Prompt 04 folds existing test-debt
rows into WS-0 when a new P4 round is planned.

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Characterization test suite | `app/src/test/.../characterization/` in target repo | WS-0 exit gate; §2 BEFORE/AFTER columns; Tier 2/3 runs |
| Smoke-flow list (P0/P1 table) | `PROJECT_STATE.md` (pinned Backlog block) | `07-build-verification.md` Tier 3; prompts 15/16; `07-checklists/release-readiness.md` |
| Degraded-gate annotations | `docs/factory/CHANGELOG.md` entries | Prompt 15 evidence review |
| Test-debt rows | `PROJECT_STATE.md` Backlog | Prompt 04 next-round WS-0 scoping |
