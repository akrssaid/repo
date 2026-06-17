# R8 & Shrinking — Enablement, Keep Rules, Mapping Lifecycle

This is the single method for code shrinking, obfuscation, and resource shrinking. The
enablement decision lands in the P4 roadmap; keep-rule work is verified at Tier 3
(`02-engineering/07-build-verification.md` — manifest/ProGuard changes are always Tier 3); the
log-stripping recipe in §6 is what `07-checklists/release-readiness.md` requires; the mapping
lifecycle in §7 is what `10-releases/release-workflow.md` stage 5 deobfuscates with. Before
enabling R8 on a previously-unshrunk app, the safety net of `02-engineering/08-testing-strategy.md`
§2 (row "R8 enablement") must be in place.

## 1. Enablement Decision

The modern default posture — and the factory's target state for every shipped app — is:

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true   // requires isMinifyEnabled
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro",
            )
        }
        debug { isMinifyEnabled = false }  // never shrink debug; it kills iteration speed
    }
}
```

| Current state | Decision |
|---|---|
| Already `minifyEnabled true` and shipping | Keep; audit rules with §3's verification loop (legacy `proguard-rules.pro` files accumulate dead `-keep class ** { *; }` blankets — tighten them) |
| Disabled, app uses reflection-heavy SDKs (Gson, legacy DI, JNI) | Enable as its own roadmap workstream item with §2–§4 executed in full; never bundle with another migration |
| Disabled, small app, deadline pressure | Enabling is still the default recommendation (size + a real obfuscation pass), but it is deferrable — record a High-severity Backlog row, not a silent skip |
| `proguard-android.txt` (non-optimize) in use | Switch to `proguard-android-optimize.txt`; re-run Tier 3 |

## 2. Reflection Inventory (run BEFORE enabling)

Everything reached by name at runtime is invisible to R8's reachability analysis and will be
stripped or renamed. Inventory all of it first:

```bash
SRC=app/src/main
grep -rn --include="*.kt" --include="*.java" -E "Class\.forName|::class\.java\.name|\.getDeclaredMethod|\.getMethod\(" $SRC
grep -rn --include="*.kt" --include="*.java" -E "TypeToken|@SerializedName|fromJson\(.*::class" $SRC   # Gson
grep -rn --include="*.kt" --include="*.java" -E "@JavascriptInterface" $SRC                            # WebView bridges
grep -rn --include="*.kt" --include="*.java" -E "external fun|System\.loadLibrary|native " $SRC        # JNI surface
grep -rn --include="*.xml" -E "android:name=\"[a-z]" $SRC/res app/src/main/AndroidManifest.xml         # class refs in XML
grep -rn -E "RegisterNatives|JNIEXPORT" app/src/main/cpp 2>/dev/null                                   # native-side registrations
```

```powershell
$src = "app\src\main"
Get-ChildItem $src -Recurse -Include *.kt,*.java | Select-String -Pattern 'Class\.forName|::class\.java\.name|\.getDeclaredMethod|\.getMethod\('
Get-ChildItem $src -Recurse -Include *.kt,*.java | Select-String -Pattern 'TypeToken|@SerializedName|fromJson\(.*::class'
Get-ChildItem $src -Recurse -Include *.kt,*.java | Select-String -Pattern '@JavascriptInterface'
Get-ChildItem $src -Recurse -Include *.kt,*.java | Select-String -Pattern 'external fun|System\.loadLibrary|native '
Get-ChildItem $src -Recurse -Include *.xml | Select-String -Pattern 'android:name="[a-z]'
if (Test-Path app\src\main\cpp) { Get-ChildItem app\src\main\cpp -Recurse | Select-String -Pattern 'RegisterNatives|JNIEXPORT' }
```

Record hits as a table (class → why reflective → rule planned) in the CHANGELOG entry or the
engineering audit; this table is the checklist §3 works through.

## 3. Keep-Rule Authoring Loop

Write **targeted** rules per inventory hit — never blanket `-keep class com.example.** { *; }`
(it disables shrinking for your whole app and hides real problems). Loop per hit:

1. **Write the narrowest rule** that covers the reflective access, in `app/proguard-rules.pro`:

```proguard
# Gson model package: keep fields (names are the wire format), allow class-name obfuscation
-keepclassmembers class com.example.app.data.model.** { <fields>; }
# JNI: native methods' names and containing classes are looked up from C++
-keepclasseswithmembers class * { native <methods>; }
# WebView JS bridge: JS calls methods by name
-keepclassmembers class com.example.app.web.JsBridge { @android.webkit.JavascriptInterface <methods>; }
```

2. **Build with diagnostics enabled** — add once to `proguard-rules.pro`:

```proguard
-printusage build/outputs/r8/usage.txt     # what R8 REMOVED
-printseeds build/outputs/r8/seeds.txt     # what your -keep rules matched (entry points)
-printconfiguration build/outputs/r8/configuration.txt  # the merged config incl. all library rules
```

```bash
./gradlew :app:assembleRelease
```

3. **Verify**: the reflective class must NOT appear in `usage.txt` (removed) and MUST appear in
   `seeds.txt` (your rule matched something — an unmatched rule is silently inert). Check rename
   status in `app/build/outputs/mapping/release/mapping.txt`:

```bash
grep -E "^com\.example\.app\.data\.model" app/build/outputs/mapping/release/mapping.txt | head -20
```

```powershell
Select-String -Path app\build\outputs\mapping\release\mapping.txt -Pattern '^com\.example\.app\.data\.model' | Select-Object -First 20
```

   A kept-but-renamed class shows `com.example.X -> a.b:`; fields you kept must map to themselves.
4. **On-device test of the obfuscated build** — the only proof that counts: install the release
   APK and run the P0 smoke flows (`08-testing-strategy.md` §5), watching for
   `ClassNotFoundException`, `NoSuchMethodError`, fields arriving null after JSON parse, blank
   WebViews, and `UnsatisfiedLinkError`. Procedure and crash-grep per
   `07-build-verification.md` Emulator Smoke.

Remove the three `-print*` lines or leave them (they only write build-dir files); never ship
`-dontobfuscate`/`-dontshrink` "temporarily" — that is the decision in §1, not a workaround.

## 4. Full Mode vs Compat Mode

R8 **full mode is the default since AGP 8.0** (the `android.enableR8.fullMode` flag is gone in
current AGP). Full mode is stricter than the old compat mode: it does not keep default
constructors of un-kept classes, ignores `Class.forName(...).newInstance()` patterns compat mode
special-cased, and aggressively removes unused interface implementations.

| Situation | Rule |
|---|---|
| AGP 8.x (factory baseline) | Full mode, no opt-out exists — author rules to full-mode strictness |
| AGP 7.x legacy app, R8 already on | Expect breakage on the AGP 8 bump: re-run §3's loop as part of `migrations/agp-migration.md` Tier 3 |
| Library ships consumer rules written for compat mode | Symptom: release-only crash in SDK internals after AGP bump → add the SDK vendor's updated rules (check the SDK's release notes) before writing your own |
| Gson + full mode | The classic casualty: full mode strips no-arg constructors → Gson silently uses `Unsafe` allocation → fields null. The §5 Gson rules are mandatory, not optional |

## 5. Baseline Keep-Rule Sets per Common SDK

Modern AARs ship consumer rules (`META-INF/proguard/`) that R8 merges automatically — check
`-printconfiguration` output before adding rules that already exist. The table is what you add
**on top** for app-side usage:

| SDK | App-side rules needed | Why |
|---|---|---|
| Gson | `-keepclassmembers` fields on model packages (see §3); `-keep class * extends com.google.gson.reflect.TypeToken` and `-keepattributes Signature` for generic `TypeToken` use | Pure reflection; field names = wire format; generics need `Signature` attribute |
| Retrofit + OkHttp | Ship good consumer rules; you only keep your **model classes** (per your converter — Gson/Moshi rules above) and `-keepattributes Signature, *Annotation*` if not already present | Retrofit reflects on interface annotations + generic return types |
| kotlinx.serialization | **Usually none.** The compiler plugin generates serializer code at compile time — no runtime reflection on your models, and the runtime ships consumer rules. Exception: `serializer()` lookups on classes only referenced by name (rare polymorphic-by-classname setups) | This is why it is the factory-preferred serializer for shrunk apps (`08-knowledge/android/modern-stack.md`) |
| WebView JS bridges | `-keepclassmembers` + `@JavascriptInterface` pattern (§3); `-keepattributes JavascriptInterface` | JS calls Java methods by string name |
| JNI | `-keepclasseswithmembers class * { native <methods>; }`; also keep any class/method called **from** C++ via `FindClass`/`GetMethodID` (inventory step §2) | Symbol lookup by name in both directions |
| Play Core / in-app updates / split install | Add the Play Core proguard rules from the library docs if using `SplitCompat`; feature-delivery reflection targets | Dynamic-delivery loads classes reflectively |

## 6. Log Stripping with `-assumenosideeffects`

`07-checklists/release-readiness.md` requires debug logging gone from release. The R8 recipe:

```proguard
# proguard-rules.pro — release only (file is only applied to minified builds anyway)
-assumenosideeffects class android.util.Log {
    public static int v(...);
    public static int d(...);
    public static int i(...);
}
```

Caveats: (1) works only with R8 optimization ON (`proguard-android-optimize.txt`); (2) the
*arguments* are not removed — an expensive `"state=" + bigObject.dump()` still executes; guard
hot-path logs with a `BuildConfig.DEBUG` check instead of relying on this; (3) keep `Log.w`/
`Log.e` — release diagnostics depend on them; (4) verify post-build:

```bash
adb logcat -d | grep -E "D/|V/" | grep -i "<app-tag>"      # expect zero app-tag hits during smoke
```

```powershell
adb logcat -d | Select-String -Pattern "D/|V/" | Select-String -Pattern "<app-tag>"
```

## 7. mapping.txt Lifecycle

Every release build writes `app/build/outputs/mapping/release/mapping.txt` (plus `seeds.txt`,
`usage.txt`, `configuration.txt`). The mapping is **unique per build** — a rebuilt artifact has a
new mapping; the archived file must be the one from the exact uploaded AAB.

1. **Archive per release** (Tier 3 output, before the AAB leaves the machine):

```bash
mkdir -p docs/factory/releases/v<versionName>
cp app/build/outputs/mapping/release/mapping.txt docs/factory/releases/v<versionName>/mapping.txt
```

```powershell
New-Item -ItemType Directory -Force docs\factory\releases\v<versionName>
Copy-Item app\build\outputs\mapping\release\mapping.txt docs\factory\releases\v<versionName>\mapping.txt
```

2. **Play Console upload**: AAB uploads carry the mapping automatically inside the bundle
   tooling flow when built by AGP ≥ 4.2; confirm in Play Console → App bundle explorer →
   the version → Downloads tab shows "ReTrace mapping file". If absent, upload manually
   (App integrity / deobfuscation files). AppGallery/RuStore: upload via their crash-service
   consoles if their crash SDK is integrated; otherwise the local archive is the only copy.
3. **Retrace** a production stack trace (release-workflow stage 5):

```bash
# retrace ships in cmdline-tools; stacktrace.txt = the raw obfuscated trace from the console
$ANDROID_HOME/cmdline-tools/latest/bin/retrace docs/factory/releases/v<versionName>/mapping.txt stacktrace.txt
```

```powershell
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\retrace.bat" docs\factory\releases\v<versionName>\mapping.txt stacktrace.txt
```

Losing a release's mapping.txt makes its crashes permanently unreadable — treat the archive copy
as a release-readiness blocker (`07-checklists/release-readiness.md`).

## 8. Resource Shrinking & keep.xml

`isShrinkResources = true` removes resources unreferenced after code shrinking. Resources
referenced **dynamically** (`resources.getIdentifier("ic_" + name, ...)`, animation files loaded
by name, resources used only from native code or server-driven config) get removed → runtime
`Resources$NotFoundException` or silent blank UI. Declare them:

```xml
<!-- app/src/main/res/raw/keep.xml -->
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@drawable/ic_badge_*, @raw/sound_*"
    tools:discard="@layout/unused_legacy_screen" />
```

Find dynamic lookups during §2's inventory: grep `getIdentifier` (both shells, same patterns as
§2). Shrinker decisions are logged at `app/build/outputs/mapping/release/resources.txt` — check
it when a resource goes missing. Strict mode (`tools:shrinkMode="strict"`) only after the keep
list is proven over a full smoke pass.

## 9. Troubleshooting

| Symptom (release build only) | Likely missing keep | Diagnose with |
|---|---|---|
| `ClassNotFoundException` / `NoSuchMethodError` at launch or feature entry | Class reached via `Class.forName`/XML name | `grep "<class>" usage.txt` (was it removed?); then mapping.txt (renamed?) |
| JSON fields all null / defaults, no crash | Gson models stripped of fields or no-arg ctor (full mode) | mapping.txt: are model field names mapped to themselves? §5 Gson rules |
| `kotlinx.serialization` `SerializationException: Serializer for class X not found` | Polymorphic by-name lookup on a removed serializer | seeds.txt for the serializer class; add targeted `-keep` on the sealed hierarchy |
| `UnsatisfiedLinkError` / native crash in `GetMethodID` | JNI-called method renamed/removed | §5 JNI rules; grep the C++ for the method string |
| WebView button does nothing | `@JavascriptInterface` method renamed | mapping.txt for the bridge class; §5 WebView rules |
| `Resources$NotFoundException` or missing image | Resource shrinker removed a dynamic resource | `resources.txt` shrinker log; add to `keep.xml` (§8) |
| Crash inside an SDK's obfuscated internals after AGP bump | SDK rules written for compat mode (§4) | `-printconfiguration` — are the SDK's consumer rules present/current? |
| APK barely smaller after enablement | A blanket `-keep ** { *; }` neutering R8 | `seeds.txt` size; grep `proguard-rules.pro` for `**` keeps |
| Works on fresh install, crashes after update | Not R8 — data migration; hand off | `02-engineering/11-data-migration-safety.md` |

## Outputs & Where They Feed

| Output | Lives at | Feeds |
|---|---|---|
| Reflection inventory table | CHANGELOG entry / `audits/ENGINEERING_AUDIT.md` | §3 loop; Tier 3 evidence |
| `proguard-rules.pro` (targeted rules + log stripping) | target repo `app/` | `07-checklists/release-readiness.md`; Tier 3 |
| Archived `mapping.txt` per release | `docs/factory/releases/v<versionName>/` | Release-workflow stage 5 retrace; `09-templates/release-report.md` |
| `keep.xml` | `app/src/main/res/raw/` | Resource-shrink safety; size baseline (`07-build-verification.md`) |
