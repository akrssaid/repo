# Material 3 Modernization Playbook

This playbook defines how to move a target app to Material 3 (Material You): gap assessment,
theme migration for XML and Compose, dynamic color with brand fallback, tonal palette generation,
component modernization order, typography migration, and the pitfalls that sink mixed-toolkit
apps. Consumed by `01-core/prompts/07-design-redesign.md` (to scope the redesign),
`08-implementation-handover.md` (to spec the work), and `09-ui-implementation.md` (to execute it).
Background concepts live in `08-knowledge/design/material3-essentials.md`; this doc is the
operational sequence. Verification is per-screen visual diff plus
`07-checklists/design-modernization.md`.

## Step 1 — Gap Assessment

Determine where the app stands. Inputs: toolkit column of
`docs/factory/audits/SCREEN_INVENTORY.md` and the extraction output from
`03-design/04-design-system-extraction.md`.

```powershell
rg -n "Theme.MaterialComponents|Theme.AppCompat|Theme.Material3" --glob "**/res/values*/*.xml"
rg -n "com.google.android.material:material" --glob "**/*.gradle*" --glob "**/libs.versions.toml"
rg -n "androidx.compose.material:material\b|androidx.compose.material3" --glob "**/*.gradle*" --glob "**/libs.versions.toml"
rg -c "androidx.compose.material\.(?!icons)" --type kotlin --pcre2
```

| Starting point | Migration path |
|---|---|
| `Theme.AppCompat.*` parents | Two hops: AppCompat → MaterialComponents bridge → M3. Budget XL effort; do the bridge step as its own verified commit |
| `Theme.MaterialComponents.*` | Direct M3 theme migration (Step 2). M for theme + per-component passes |
| XML `Theme.Material3.*` already | Token/dynamic-color cleanup only |
| Compose M2 (`androidx.compose.material`) | API migration (Step 3) |
| Compose M3 already | Verify scheme completeness + dynamic color |
| Mixed XML + Compose | Do BOTH paths and bind them (Pitfall 2) |

Record the path per screen group in the implementation handover
(`05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md`).

## Step 2 — XML Path: MaterialComponents → Material3

1. Ensure `com.google.android.material:material` ≥ 1.12.0 (verify latest stable; June 2026
   baseline ~1.13.x) in the version catalog.
2. Change theme parents in `res/values/themes.xml` and `res/values-night/themes.xml`:

   | From | To |
   |---|---|
   | `Theme.MaterialComponents.DayNight.NoActionBar` | `Theme.Material3.DayNight.NoActionBar` |
   | `Theme.MaterialComponents.Light.*` | `Theme.Material3.Light.*` |
   | `ThemeOverlay.MaterialComponents.*` | `ThemeOverlay.Material3.*` |
   | `Widget.MaterialComponents.Button` | `Widget.Material3.Button` |
   | `Widget.MaterialComponents.Toolbar` | `Widget.Material3.Toolbar` |
   | `TextAppearance.MaterialComponents.Headline6` | `TextAppearance.Material3.TitleLarge` |

3. Map/define color attributes. M3 expands the role set — fill at least these in both light and
   night themes (values from the tonal palette, Step 5):

   | M2 attribute | M3 attribute(s) | Notes |
   |---|---|---|
   | `colorPrimary` | `colorPrimary` + `colorOnPrimary` + `colorPrimaryContainer` + `colorOnPrimaryContainer` | Container roles are new — components default to them |
   | `colorPrimaryVariant` | (removed) | Replace usages with `colorPrimaryContainer` or a tonal value |
   | `colorSecondary` | `colorSecondary` + on/container variants | |
   | (none) | `colorTertiary` + variants | New role; derive from seed |
   | `colorSurface` | `colorSurface`, `colorSurfaceVariant`, `colorSurfaceContainer{Lowest…Highest}` | Surface containers replace elevation-shadow hierarchy |
   | `colorOnSurface` | `colorOnSurface`, `colorOnSurfaceVariant`, `colorOutline`, `colorOutlineVariant` | |
   | `colorError` | `colorError` + on/container variants | |

4. Sweep widget styles: `rg -n "MaterialComponents" --glob "**/res/**/*.xml"` must return zero
   after migration (excluding comments).
5. Build and launch; M3 themes throw `IllegalArgumentException`/inflate errors on missing required
   attributes — fix until every inventoried screen opens.

## Step 3 — Compose Path: M2 → M3

1. Dependency swap in the version catalog (BOM keeps versions aligned):

   ```toml
   compose-bom = { group = "androidx.compose", name = "compose-bom", version = "2025.06.00" }
   material3 = { group = "androidx.compose.material3", name = "material3" }
   ```

   Remove `androidx.compose.material:material`; keep `androidx.compose.material:material-icons-extended`
   (icons remain in the M2 artifact namespace) if used.
2. Mechanical import rewrite: `androidx.compose.material.*` → `androidx.compose.material3.*`,
   then fix API diffs:

   | M2 API | M3 API | Diff |
   |---|---|---|
   | `MaterialTheme(colors = …)` | `MaterialTheme(colorScheme = …)` | `Colors` → `ColorScheme`; `lightColors()` → `lightColorScheme()` |
   | `Scaffold` (with `drawerContent`) | `Scaffold` + `ModalNavigationDrawer` | Drawer moved out of Scaffold |
   | `TopAppBar` | `TopAppBar` / `CenterAlignedTopAppBar` / `MediumTopAppBar` / `LargeTopAppBar` | Pick variant; needs `TopAppBarDefaults.*ScrollBehavior` for scroll-collapse |
   | `BottomNavigation` / `BottomNavigationItem` | `NavigationBar` / `NavigationBarItem` | Renamed |
   | `Backdrop*`, `BottomDrawer` | (removed) | Redesign with sheets |
   | `AlertDialog(buttons = …)` | `AlertDialog(confirmButton = …, dismissButton = …)` | Slot API changed |
   | `ModalBottomSheetLayout` | `ModalBottomSheet` | Composable, not layout wrapper |
   | `Typography(h1 … caption)` | `Typography(displayLarge … labelSmall)` | Full type-scale rename (Step 6) |
   | `MaterialTheme.colors.primary` | `MaterialTheme.colorScheme.primary` | Sweep with `rg -n "MaterialTheme\.colors\." --type kotlin` |

3. After the rewrite, `rg -n "androidx\.compose\.material\.(?!icons)" --type kotlin --pcre2` must
   return zero hits.

## Step 4 — Dynamic Color with Brand Fallback

Default policy: enable dynamic color on Android 12+ (API 31+), fall back to the brand scheme below.
Override only with a recorded decision in `docs/factory/PROJECT_STATE.md` (e.g., brand color is
legally mandated, or monetization surfaces depend on a specific accent).

XML (in `Application.onCreate`):

```kotlin
DynamicColors.applyToActivitiesIfAvailable(this)
```

Compose theme:

```kotlin
@Composable
fun AppTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    val context = LocalContext.current
    val colorScheme = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        darkTheme -> DarkBrandColorScheme
        else -> LightBrandColorScheme
    }
    MaterialTheme(colorScheme = colorScheme, typography = AppTypography, content = content)
}
```

Brand-fallback strategy: the brand seed color generates the static schemes (Step 5) used below
API 31 and wherever dynamic color is disabled. Surfaces that must stay on-brand regardless of
wallpaper (paywall hero, store-screenshot frames) use explicit brand tokens, not `colorScheme`
roles — document each such exception in the design handover.

## Step 5 — Tonal Palette from Brand Seed (Material Theme Builder)

1. Identify the brand seed color: the dominant brand color from the color census
   (`03-design/04-design-system-extraction.md`) or the app icon's primary hue.
2. Open Material Theme Builder (https://material-foundation.github.io/material-theme-builder/),
   set the seed as Primary; adjust secondary/tertiary only if the brand defines them.
3. Export **both**: "Android Views (XML)" → `colors.xml` + `themes.xml` snippets, and
   "Jetpack Compose" → `Color.kt` + `Theme.kt`. Use the export matching the app's toolkit; Mixed
   apps take both and keep the hex values identical (Pitfall 2).
4. Commit the generated files verbatim, then wire theme attributes to them. Never hand-pick tonal
   steps — regenerating from seed must be repeatable.
5. Validate contrast of the generated scheme on real surfaces: on-color over color pairs must hit
   ≥ 4.5:1 for body text (`07-checklists/accessibility.md`); if the seed produces a failing
   primary (common with bright yellows/light brand colors), use the Theme Builder's adjusted tone
   and record the delta from the raw brand hex.

## Step 6 — Typography Migration to M3 Type Scale

Map existing sizes (sp census from `03-design/04-design-system-extraction.md`) onto the 15-role M3
scale; collapse off-scale sizes to the nearest role:

| M3 role | Default | Typical legacy mapping |
|---|---|---|
| displayLarge/Medium/Small | 57/45/36sp | splash numerals, hero stats |
| headlineLarge/Medium/Small | 32/28/24sp | H4/H5, screen titles |
| titleLarge/Medium/Small | 22/16/14sp | H6/Subtitle1/Subtitle2, toolbar titles, card titles |
| bodyLarge/Medium/Small | 16/14/12sp | Body1/Body2/Caption |
| labelLarge/Medium/Small | 14/12/11sp | Button/overline, chips, nav labels |

Rules: keep the custom brand font by overriding `fontFamily` per role, not per usage; all sizes in
sp; every text style in the app must resolve to a role (zero literal `textSize`/`fontSize` outside
the theme when done — re-run the Pass 2 census to verify).

## Step 7 — Component Modernization Priority Order

Work strictly in this order; each tier is a separate commit + visual-diff checkpoint:

1. **Theme** (Steps 2–6) — everything downstream inherits it.
2. **Buttons & text fields** — highest interaction density; M3 button variants (filled, tonal,
   outlined, text, elevated) replace custom button styles flagged "replace-with-Material" in the
   extraction; `TextInputLayout`/`OutlinedTextField` to M3 styling.
3. **App bars & navigation** — top app bar variant per screen, `NavigationBar`/NavigationRail,
   predictive-back-friendly transitions; edge-to-edge insets handled here (mandatory at
   targetSdk 35+).
4. **Cards & lists** — M3 card variants (elevated/filled/outlined), list item spec, surface
   container colors instead of shadow stacking.
5. **Dialogs & sheets** — M3 `AlertDialog`/`MaterialAlertDialogBuilder`, `ModalBottomSheet`,
   full-screen dialogs for complex tasks.

Within each tier, process screens by traffic priority (H → M → L) from the inventory.

## Common Pitfalls

| # | Pitfall | Symptom | Prevention/Fix |
|---|---|---|---|
| 1 | Mixed M2/M3 artifacts on the classpath | Two button styles in one screen; crashes like "must use Theme.Material3" | Gradle dependency sweep after Step 3; ban `androidx.compose.material:material` via lint/Konsist rule |
| 2 | XML ↔ Compose theme drift in Mixed apps | Dialog (XML) and screen (Compose) show different primaries | Single source of truth: generate both exports from one seed (Step 5); in Compose-in-XML screens verify the host activity theme is M3 |
| 3 | Dynamic color contrast failures | Brand-adjacent wallpaper yields illegible CTA | Use on-color roles exclusively (never brand hex on dynamic surfaces); test 3 wallpapers + both themes during verification |
| 4 | `colorPrimaryVariant` orphans | Build fine, but views fall back to black/teal | `rg -n "colorPrimaryVariant|colorSecondaryVariant"` must be zero post-migration |
| 5 | Elevation overlays vs tonal surfaces | Dark theme cards invisible or double-tinted | Replace `app:cardElevation` hierarchies with `colorSurfaceContainer*` roles (tier 4) |
| 6 | Icon tint regressions | Icons invisible after theme swap | Tint with `?attr/colorOnSurfaceVariant` / `LocalContentColor`, never literal `@color/black` |

## Verification

1. **Per-screen visual diff**: re-capture every SCR-ID/state into
   `docs/factory/assets/screenshots/redesigned/` with filenames mirroring `baseline/`
   (`03-design/03-screenshot-capture.md`); review pairs side by side; every intentional change
   maps to a handover spec item, every unintentional change is a defect.
2. **Sweeps return zero**: `MaterialComponents` in res, `MaterialTheme.colors.`,
   `androidx.compose.material.` (non-icons), `colorPrimaryVariant`.
3. **Matrix run**: light/dark × dynamic-color on/off (API 31+ emulator + API 26 emulator for
   fallback path) × font scale 1.0/1.5 on traffic-H screens.
4. **Gate**: complete `07-checklists/design-modernization.md` and the contrast/touch-target items
   of `07-checklists/accessibility.md`; record completion in `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md` per `01-core/prompts/09-ui-implementation.md`.
