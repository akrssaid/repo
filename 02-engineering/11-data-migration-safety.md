# Data Migration Safety — Schemas, Prefs, Upgrade Paths

User data is the one thing a release cannot roll back. This method owns every procedure that
protects it: Room schema migrations and their tests, SharedPreferences→DataStore correctness,
the upgrade-path install test, file-format versioning, and the interaction with Android backup.
It is the procedure behind `10-releases/release-workflow.md` stage 6.3 (data-migration
reversibility pre-flight), the migration test that stage 6.3 and Tier 3 cite, and the §5 install
test that prompts 15/16 execute. Safety-net prerequisites per `02-engineering/08-testing-strategy.md`
§2; verdicts/scales per `01-core/CONVENTIONS.md`.

## 1. Room Schema Export (prerequisite for everything else)

Migrations cannot be tested without exported schema JSONs. If the target app lacks them, wiring
this is the **first** data-touching change of P4 — before any schema change ships:

```kotlin
// app/build.gradle.kts — KSP arg (factory baseline is KSP, never kapt)
ksp { arg("room.schemaLocation", "$projectDir/schemas") }

android {
    sourceSets.getByName("androidTest").assets.srcDir("$projectDir/schemas") // for MigrationTestHelper
}
```

```kotlin
@Database(entities = [Doc::class], version = 7, exportSchema = true)  // exportSchema=true is the default — never set it false
abstract class AppDb : RoomDatabase()
```

Build once; commit `app/schemas/<db-class>/<version>.json` to git **forever** — these files are
the only record of what shipped. If old versions were never exported, reconstruct what you can:
pull the DB from an old release APK installed on an emulator (§5 archive) and inspect with
`sqlite3 .schema`, then export from that version's code if the tag still exists.

## 2. Writing a Migration

```kotlin
val MIGRATION_6_7 = object : Migration(6, 7) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // SQLite ALTER TABLE is limited: ADD COLUMN yes; drop/retype = recreate-and-copy
        db.execSQL("ALTER TABLE Doc ADD COLUMN starred INTEGER NOT NULL DEFAULT 0")
        db.execSQL("CREATE INDEX IF NOT EXISTS index_Doc_starred ON Doc(starred)")
    }
}

Room.databaseBuilder(ctx, AppDb::class.java, "app.db")
    .addMigrations(MIGRATION_6_7)        // register EVERY migration object
    .build()
```

Rules: write raw SQL matching what Room expects for the new version **exactly** (column
affinity, NOT NULL, defaults, indices — Room validates the resulting schema at open and throws
`IllegalStateException: Migration didn't properly handle` on any mismatch; the exception message
contains the expected-vs-found table diff — read it, don't guess). For drop/retype/rename on
minSdk < 30, use the create-new → `INSERT INTO new SELECT ... FROM old` → drop → rename dance,
or Room's `AutoMigration` with `@RenameColumn`/`@DeleteColumn` specs for the simple cases.

**The all-paths rule:** every schema version that **ever shipped to users** must have a migration
chain to the current version. Users update from any old version, not just N−1; a missing link
crashes (or destroys data, §3) for exactly the users who waited longest. Verify the chain
covers `min(shipped versions recorded in PROJECT_STATE Release History) → current`. Chained
single-step migrations (5→6→7) are fine; Room composes them.

## 3. Destructive-Migration Policy

`fallbackToDestructiveMigration()` (and `...OnDowngrade()`) deletes the user's database whenever
a migration path is missing. **This is a data-loss decision, not a coding convenience**:

- An agent may NEVER add a destructive fallback on its own. Finding one already present in a
  legacy app = High-severity ENG finding + Known Risk row in `PROJECT_STATE.md`.
- Shipping any release with a destructive fallback in effect requires an explicit operator
  decision recorded in the Decision Log (this is escalation trigger territory per
  `01-core/CLAUDE_MASTER.md` — destructive operations).
- Consequence either way: once version N's schema is live, **rollback is forward-fix-only**
  (release-workflow stage 6.3): N−1's code cannot open N's database, so a pulled release strands
  updated users. The release handover must say so verbatim.
- `fallbackToDestructiveMigrationOnDowngrade()` alone is tolerable (downgrades only occur for
  sideloading testers), but still recorded.

## 4. SharedPreferences → DataStore Correctness

The migration guide for the code change is the modern-stack material; the **correctness
procedure** is here:

```kotlin
val Context.settings: DataStore<Preferences> by preferencesDataStore(
    name = "settings",
    produceMigrations = { ctx ->
        listOf(SharedPreferencesMigration(ctx, "legacy_prefs_name"))   // EXACT legacy file name
    },
)
```

Semantics you must verify, not assume:

1. **One-shot and self-cleaning**: the migration runs on first DataStore read, copies all keys
   (or the subset given via `keysToMigrate`), then **deletes the SharedPreferences file**. Any
   remaining legacy read path (`getSharedPreferences("legacy_prefs_name", ...)`) now sees empty
   defaults — grep for stragglers before shipping:

```bash
grep -rn --include="*.kt" --include="*.java" "getSharedPreferences\|PreferenceManager.getDefault" app/src/main
```

```powershell
Get-ChildItem app\src\main -Recurse -Include *.kt,*.java | Select-String -Pattern 'getSharedPreferences|PreferenceManager.getDefault'
```

2. **Process death mid-migration is safe by design** — the migration commits atomically with the
   DataStore write and the prefs file is deleted only after success, so a kill mid-flight
   re-runs it next launch. What is NOT safe: two DataStores migrating the same prefs file
   (second one finds it deleted), or migrating a file that another process still writes.
3. **Type fidelity**: SharedPreferences `Set<String>` ↔ `stringSetPreferencesKey`, and a key
   stored as Int arrives as Int — a `longPreferencesKey` read of it returns null. Map every key
   to the exact legacy type, then convert in app code.
4. **Verification**: characterization test populating a real prefs file then asserting every
   migrated value via `runTest { ctx.settings.data.first() }` (instrumented, or Robolectric),
   plus the §5 upgrade-path test where the previous release wrote the prefs organically.

## 5. Upgrade-Path Install Test (canonical procedure)

The single highest-value data test: install the **previous shipped release**, create data, update
in place, verify. Prompts 15/16 and Tier 3 (`07-build-verification.md`) execute exactly this.
Prerequisite: archive every shipped release APK/AAB under `docs/factory/releases/v<versionName>/`
(same home as mapping.txt, `09-r8-shrinking.md` §7). Both builds must be signed with the same key
or `adb install -r` fails `INSTALL_FAILED_UPDATE_INCOMPATIBLE` — use the upload/debug-release
keystore consistently.

```bash
PKG=<applicationId>
OLD=docs/factory/releases/v<prev>/app-release.apk
NEW=app/build/outputs/apk/release/app-release.apk

adb uninstall $PKG || true                      # clean slate
adb install $OLD
adb shell monkey -p $PKG -c android.intent.category.LAUNCHER 1
# >>> MANUAL: exercise data-creating flows — create a doc, change 2-3 settings,
#             log in if applicable, leave something in every persisted store <<<
adb shell am force-stop $PKG

adb install -r $NEW                             # UPDATE in place — never uninstall first
adb logcat -c
adb shell monkey -p $PKG -c android.intent.category.LAUNCHER 1
sleep 8
adb logcat -d *:E | grep -E "FATAL|Migration|SQLite|IllegalStateException"
# >>> MANUAL: verify the doc opens with edits intact, settings retained, session valid <<<
```

```powershell
$pkg = "<applicationId>"
$old = "docs\factory\releases\v<prev>\app-release.apk"
$new = "app\build\outputs\apk\release\app-release.apk"

adb uninstall $pkg
adb install $old
adb shell monkey -p $pkg -c android.intent.category.LAUNCHER 1
# >>> MANUAL: exercise data-creating flows as above <<<
adb shell am force-stop $pkg

adb install -r $new
adb logcat -c
adb shell monkey -p $pkg -c android.intent.category.LAUNCHER 1
Start-Sleep -Seconds 8
adb logcat -d *:E | Select-String -Pattern "FATAL|Migration|SQLite|IllegalStateException"
# >>> MANUAL: verify data intact <<<
```

Pass = zero matches in the crash grep AND all manually created data intact. Run once from the
immediately previous release always; from the **oldest still-significant shipped version**
(Release History installs data) whenever the release contains any schema/prefs/file-format
change. Record the result in the verification report (prompt 15).

## 6. Room Migration Test (instrumented)

`MigrationTestHelper` opens a real SQLite DB at an old schema (from the §1 exported JSONs),
runs your migrations, and validates the result against the current schema:

```kotlin
@RunWith(AndroidJUnit4::class)
class DbMigrationTest {
    @get:Rule val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDb::class.java,
    )

    @Test fun migrate6To7_preservesRowsAndDefaults() {
        helper.createDatabase("test.db", 6).apply {
            execSQL("INSERT INTO Doc (id, title) VALUES (1, 'kept')")   // v6 shape — no `starred`
            close()
        }
        val db = helper.runMigrationsAndValidate("test.db", 7, true, MIGRATION_6_7)
        db.query("SELECT title, starred FROM Doc WHERE id = 1").use { c ->
            assertTrue(c.moveToFirst())
            assertEquals("kept", c.getString(0))
            assertEquals(0, c.getInt(1))                                 // DEFAULT applied
        }
    }

    @Test fun fullChain_oldestShippedToCurrent() {
        helper.createDatabase("test.db", 3).close()                      // oldest shipped version
        helper.runMigrationsAndValidate("test.db", 7, true, *ALL_MIGRATIONS)
    }
}
```

Dependency: `androidTestImplementation "androidx.room:room-testing:<room-version>"`. The
full-chain test enforces §2's all-paths rule mechanically. These tests are the "migration test
against a previous-version database snapshot" that release-workflow stage 6.3 requires — a
schema-change release without them fails pre-flight.

## 7. File-Format / Proto Versioning Rule

Any custom persisted format (JSON cache files, proto-datastore messages, binary exports) follows
the same discipline as DB schemas: (1) a version marker inside the format (proto: add fields,
never renumber/reuse tags; JSON: a `"v":` field) ; (2) readers accept **all shipped versions**
(unknown-field tolerance pinned by a characterization test, `08-testing-strategy.md` §1);
(3) writers may emit only the newest version; (4) a format change in a release = the §5 oldest-
version install test + a forward-fix-only note in the release handover, exactly like a schema
change.

## 8. Backup Rules Interaction

Auto Backup restores app data on reinstall — which means a **fresh install may wake up with
year-old data** and immediately run your full migration chain. Implications:

- `android:dataExtractionRules` (targetSdk ≥ 31; keep `android:fullBackupContent` alongside for
  API ≤ 30) controls what is backed up. Audit it in `02-engineering/06-manifest-audit.md`; this
  doc owns the migration consequence.
- DB files and SharedPreferences are backed up by default → the all-paths rule (§2) and the
  prefs-migration one-shot (§4) must tolerate restored stale state. The §4 migration re-runs
  harmlessly if the restored snapshot predates the DataStore file — verify that case once.
- Never exclude the DB to "dodge" migrations — that silently deletes user data on device
  changes. Exclude only caches, device-bound tokens (`exclude` + `cloud-backup`/`device-transfer`
  split), and anything the data-safety listing claims is not stored.
- Test: `adb shell bmgr backupnow <pkg>` → uninstall → reinstall → verify restore + migration
  (emulator with a Google account; same commands both shells).

## 9. Pre-Release Data-Safety Checklist

Referenced by `07-checklists/release-readiness.md`; all five must hold for any release touching
persisted data:

- [ ] Every shipped schema version has a tested chain to current (§2, §6 full-chain test green).
- [ ] No destructive-migration fallback in effect without an operator Decision Log entry (§3).
- [ ] Upgrade-path install test passed from previous release (and oldest-significant on
      schema/format changes), data verified intact (§5).
- [ ] Forward-fix-only rollback consequence stated in `RELEASE_HANDOVER_v<N>.md` if any
      schema/prefs/format change ships (§3, §7).
- [ ] Backup restore path exercised or explicitly waived with reason (§8).

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Exported schema JSONs | `app/schemas/` (committed) | §6 tests; future migrations |
| Migration + full-chain tests | `app/src/androidTest/` | Tier 3; release-workflow stage 6.3 pre-flight |
| Upgrade-path test result | `reports/VERIFICATION_REPORT_v<versionName>.md` | Prompts 15/16; `07-checklists/release-readiness.md` |
| Destructive-migration decisions | `PROJECT_STATE.md` Decision Log + Known Risks | Release handover; stage 6 rollback plan |
| §9 checklist block | embedded in release-readiness run | P8 gate verdict (PASS / PASS with waivers / FAIL) |
