# Design System Extraction Method

This document defines how to extract the CURRENT design system of a target app from its code: the
actual colors, typography, spacing, shapes, elevation, icons, and component patterns in use — plus
a divergence report quantifying how much of the UI bypasses tokens with one-off values. The output
is the "Current Design System" section of the design audit and the designer's raw material in the
design handover (`05-handover/DESIGN_HANDOVER_SCHEMA.md`). Consumed by
`01-core/prompts/05-design-audit.md` and `06-design-handover.md`. Run all commands from the target
repo root with ripgrep (`rg`).

## Principles

1. **Extract what IS, not what should be.** This is forensic work; recommendations belong to the
   audit and redesign.
2. **Census, not sample.** Counts matter — "47 distinct hex values, 9 of them tokenized" is the
   sentence that justifies a token-first redesign.
3. **Cover both worlds.** Most legacy apps are Mixed toolkit; always run both the XML and Compose
   passes, and note which screens (by toolkit column in `docs/factory/audits/SCREEN_INVENTORY.md`)
   each token set governs.

## Pass 1 — Color

### Declared tokens

```powershell
rg -n "<color name=" --glob "**/res/values*/colors.xml"
rg -n "colorPrimary|colorSecondary|colorSurface|colorOnSurface|colorError|android:colorBackground" --glob "**/res/values*/themes.xml" --glob "**/res/values*/styles.xml"
rg -n "Color\(0x[0-9A-Fa-f]{8}\)|Color\(0x[0-9A-Fa-f]{6}\)" --type kotlin
rg -n "lightColorScheme\(|darkColorScheme\(|lightColors\(|darkColors\(" --type kotlin
```

Record every declared color: name, hex value, defined where (`colors.xml`, `Color.kt`), and dark
variant (`values-night/` or `darkColorScheme`) if any.

### Usage census

```powershell
rg -o "@color/\w+" --glob "**/res/**/*.xml" | sort | uniq -c | sort -rn        # via Git Bash
rg -on "#[0-9A-Fa-f]{6,8}\b" --glob "**/res/layout*/**" --glob "**/res/drawable*/**"
rg -n "Color\(0x" --type kotlin --glob "!**/ui/theme/**" --glob "!**/Color.kt"
rg -n "setBackgroundColor\(|Color\.parseColor\(" --type-add "src:*.{kt,java}" -t src
```

(PowerShell equivalent of the uniq pipeline:
`rg -o "@color/\w+" --glob "**/res/**/*.xml" | Group-Object | Sort-Object Count -Descending | Select-Object Count, Name`)

The second/third commands find **rogue colors**: hex literals in layouts/drawables and `Color(0x…)`
outside the theme package. Every rogue hit is a divergence entry.

### Color census table (output format)

| Color | Hex (light/dark) | Declared in | Uses | Rogue duplicates | Semantic role (inferred) |
|---|---|---|---|---|---|
| `colorPrimary` / `purple_500` | #6200EE / #BB86FC | colors.xml, themes.xml | 41 | #6200EE hardcoded ×3 in layouts | Brand primary |
| (unnamed) | #FF5722 | — (layout literals only) | 7 | n/a — never tokenized | Accent? CTA on SCR-004/SCR-009 |

Also record: total distinct color values found anywhere vs count reachable through the theme;
whether `values-night` exists; whether any screen sets colors programmatically.

## Pass 2 — Typography

```powershell
rg -n "<style name=\"TextAppearance|parent=\"TextAppearance" --glob "**/res/values*/*.xml"
rg -n "android:textSize=\"\d+sp\"|android:textSize=\"\d+dp\"" --glob "**/res/layout*/**"
rg -o "android:textSize=\"\d+(sp|dp)\"" --glob "**/res/layout*/**" | sort | uniq -c | sort -rn
rg -n "fontFamily|<font-family|ResourcesCompat.getFont" --glob "**/res/**" --type kotlin
rg -n "Typography\(|TextStyle\(|fontSize = \d+\.sp" --type kotlin
```

Output: the **sp-size census** — a table of every text size in use, its count, and whether it
comes from a textAppearance/Typography token or a literal:

| Size | Count | Tokenized? | Token / where hardcoded |
|---|---|---|---|
| 14sp | 38 | partial | `TextAppearance.Body1` (22) + literals in 16 layouts |
| 13sp | 4 | no | literals only — off-scale value |

Flag immediately: `textSize` in **dp** (a11y bug — breaks font scaling; feed to
`03-design/09-accessibility-review.md`), more than ~8 distinct sizes (no scale), and custom fonts
loaded without a fallback. Record font families: declared faces, weights actually used, and
whether `res/font/` XML or runtime loading is used.

## Pass 3 — Spacing & Dimens

```powershell
rg -n "<dimen name=" --glob "**/res/values*/dimens.xml"
rg -o "android:(layout_margin|padding)\w*=\"\d+dp\"" --glob "**/res/layout*/**" | sort | uniq -c | sort -rn
rg -o "\.padding\(\d+\.dp\)|Spacer\(.*\d+\.dp" --type kotlin | sort | uniq -c | sort -rn
rg -on "\b\d+\.dp\b" --type kotlin --glob "!**/ui/theme/**"
```

Output: the **dp census** — every margin/padding value with counts, split tokenized (`@dimen/`,
spacing object in theme) vs literal. Derive the implicit grid: if 80% of values are multiples of
8 dp, the app has a latent 8-dp grid; values like 7 dp, 13 dp, 22 dp are drift. Record the top 10
most-used values — they become the proposed spacing scale's anchor in the handover.

## Pass 4 — Shape & Corner Radii

```powershell
rg -n "cornerRadius|app:cardCornerRadius|shapeAppearance" --glob "**/res/**/*.xml"
rg -n "<corners android:radius" --glob "**/res/drawable*/**"
rg -n "RoundedCornerShape\(|MaterialTheme.shapes|CircleShape" --type kotlin
```

Output table: every radius value, count, and source (shapeAppearance token vs drawable literal vs
`RoundedCornerShape` literal). More than 4 distinct radii = systemic inconsistency candidate
(DES finding, class SYS — see `03-design/01-design-audit.md`).

## Pass 5 — Elevation

```powershell
rg -n "android:elevation|app:elevation|app:cardElevation|translationZ" --glob "**/res/**/*.xml"
rg -n "elevation = \d+\.dp|shadowElevation|tonalElevation" --type kotlin
```

Record distinct elevation values and whether the app relies on shadows (M2 idiom) vs tonal
elevation (M3 idiom) — directly feeds the gap assessment in
`03-design/05-material3-modernization.md`.

## Pass 6 — Iconography

```powershell
rg -l "<vector" --glob "**/res/drawable*/**"
Get-ChildItem -Recurse -Include *.png,*.webp,*.jpg -Path **\res\drawable* | Group-Object Extension | Select-Object Count, Name
rg -n "Icons\.(Default|Filled|Outlined|Rounded|Sharp|TwoTone)" --type kotlin
```

Output: vector vs raster census (count by type); icon style mix (Compose `Icons.Filled` vs
`Icons.Outlined` in the same app = inconsistency); third-party icon sets; any raster icons that
should be vectors (UI glyphs shipped as PNG). Launcher icon state (adaptive? monochrome layer?) is
recorded here but deep-dived in `03-design/06-icon-redesign.md`.

## Pass 7 — Component Patterns

```powershell
rg -n "class \w+ ?: ?\w*(View|ViewGroup|Layout|Button|TextView)\b" --type kotlin
rg -n "extends \w*(View|ViewGroup|LinearLayout|FrameLayout|ConstraintLayout)\b" --type java
rg -n "^@Composable" --type kotlin -c
rg -ln "<include layout=" --glob "**/res/layout*/**"
rg -n "<style name=\"Widget\." --glob "**/res/values*/*.xml"
```

Inventory: custom views/composables (name, what it does, screens using it), shared `Widget.*`
styles, `<include>`-reused layouts. Classify each as **keep** (genuine domain component),
**replace-with-Material** (reinvented standard component — custom button, custom dialog), or
**absorb-into-library** (candidate for `03-design/08-component-library.md`). Also detect the
component-library generation: `rg -n "com.google.android.material" --glob "**/*.gradle*"` and
note Material library version, plus AppCompat-only widgets (`rg -n "androidx.appcompat.widget"`).

## Pass 8 — Motion & Interaction Census

Extract the app's CURRENT motion and interaction reality (D14). This Pass is the raw material for the
design handover's **Motion & Interaction Inventory** section (`05-handover/DESIGN_HANDOVER_SCHEMA.md`)
and the gap input to the motion target spec in `03-design/10-motion-design.md`. As with every Pass:
census what IS, not what should be.

### Transitions, animators, and Lottie (both shells, identical patterns)

```bash
rg -n "android:animateLayoutChanges" --glob "**/res/layout*/**"
rg -ln "<set|<objectAnimator|<translate|<alpha|<scale|<rotate" --glob "**/res/anim*/**" --glob "**/res/animator*/**"
rg -ln "<transitionSet|<changeBounds|<fade|<slide|<explode|<autoTransition" --glob "**/res/transition*/**"
rg -n "TransitionManager|beginDelayedTransition|MaterialContainerTransform|MaterialSharedAxis|MaterialFadeThrough" --type kotlin --type java
rg -n "AnimatedVisibility|AnimatedContent|animate\w+AsState|updateTransition|animateContentSize|Modifier.animateItemPlacement|rememberInfiniteTransition" --type kotlin
rg -n "com.airbnb.lottie|LottieAnimation|LottieView|setAnimation\(" --type kotlin --type java --glob "**/*.xml"
```

```powershell
rg -n "android:animateLayoutChanges" --glob "**/res/layout*/**"
rg -ln "<set|<objectAnimator|<translate|<alpha|<scale|<rotate" --glob "**/res/anim*/**" --glob "**/res/animator*/**"
rg -ln "<transitionSet|<changeBounds|<fade|<slide|<explode|<autoTransition" --glob "**/res/transition*/**"
rg -n "TransitionManager|beginDelayedTransition|MaterialContainerTransform|MaterialSharedAxis|MaterialFadeThrough" --type kotlin --type java
rg -n "AnimatedVisibility|AnimatedContent|animate\w+AsState|updateTransition|animateContentSize|Modifier.animateItemPlacement|rememberInfiniteTransition" --type kotlin
rg -n "com.airbnb.lottie|LottieAnimation|LottieView|setAnimation\(" --type kotlin --type java --glob "**/*.xml"
```

Record: which screens have *any* navigation transition vs jump-cut; whether Material motion classes
(`MaterialContainerTransform`/`MaterialSharedAxis`/`MaterialFadeThrough`) are used or transitions are
ad-hoc; every Lottie/animator asset (file, where played); presence of `android:animateLayoutChanges`
(implicit, untokenized motion). Note any hardcoded durations:

```bash
rg -on "android:duration=\"\d+\"|setDuration\(\d+|durationMillis = \d+|tween\(\d+" --type kotlin --type java --glob "**/res/**/*.xml" | sort | uniq -c | sort -rn
```

```powershell
rg -on "android:duration=\"\d+\"|setDuration\(\d+|durationMillis = \d+|tween\(\d+" --type kotlin --type java --glob "**/res/**/*.xml" | Group-Object | Sort-Object Count -Descending | Select-Object Count, Name
```

Distinct duration values vs the M3 duration scale (50–600ms short/medium/long, see
`08-knowledge/design/material3-essentials.md` motion section and `03-design/10-motion-design.md`) is a
motion-divergence signal — magic durations like `250`, `400` scattered ad-hoc = no motion tokens.

### Interaction states per key component

For each component flagged **keep** or **replace-with-Material** in Pass 7, record which interaction
states are actually styled — pressed, disabled, focused, selected, hovered:

```bash
rg -ln "state_pressed|state_enabled|state_focused|state_selected|state_activated|state_checked" --glob "**/res/**/*.xml"
rg -n "rememberRipple|indication =|InteractionSource|collectIsPressedAsState|collectIsFocusedAsState|\.clickable\(" --type kotlin
rg -n "android:foreground=|selectableItemBackground|rippleColor|app:rippleColor" --glob "**/res/**/*.xml"
```

```powershell
rg -ln "state_pressed|state_enabled|state_focused|state_selected|state_activated|state_checked" --glob "**/res/**/*.xml"
rg -n "rememberRipple|indication =|InteractionSource|collectIsPressedAsState|collectIsFocusedAsState|\.clickable\(" --type kotlin
rg -n "android:foreground=|selectableItemBackground|rippleColor|app:rippleColor" --glob "**/res/**/*.xml"
```

Output an interaction-state matrix: component × {pressed, disabled, focus, selected} = styled / default /
missing. Missing focus styling on interactive elements is also an accessibility input
(`03-design/09-accessibility-review.md`).

### Gesture map

```bash
rg -n "OnSwipe|ItemTouchHelper|SwipeRefresh|swipeable|detectDragGestures|detectHorizontalDragGestures|onLongClick|setOnLongClickListener|combinedClickable|pointerInput|GestureDetector" --type kotlin --type java
rg -n "<androidx.viewpager|ViewPager2|app:layout_behavior|BottomSheetBehavior" --type kotlin --type java --glob "**/res/**/*.xml"
```

```powershell
rg -n "OnSwipe|ItemTouchHelper|SwipeRefresh|swipeable|detectDragGestures|detectHorizontalDragGestures|onLongClick|setOnLongClickListener|combinedClickable|pointerInput|GestureDetector" --type kotlin --type java
rg -n "<androidx.viewpager|ViewPager2|app:layout_behavior|BottomSheetBehavior" --type kotlin --type java --glob "**/res/**/*.xml"
```

Record each non-tap gesture: gesture (swipe-to-delete, swipe-to-refresh, long-press, drag-reorder,
pager swipe, sheet drag), where it lives (SCR-ID), and whether it has a visible alternative (an
H8 input for `03-design/01-design-audit.md`). This gesture map seeds the handover's interaction
inventory and the predictive-back assessment in `03-design/10-motion-design.md`.

## Pass 9 — Imagery & Copy Census (brief)

Two short censuses so the corresponding handover sections (Imagery & Illustration Direction;
Localization & Content Constraints) have raw material — these are lighter than Passes 1–8.

### Imagery / illustration census

```bash
rg -ln "<vector|<bitmap|app:srcCompat|android:src=" --glob "**/res/drawable*/**"
rg -n "Image\(|AsyncImage|rememberAsyncImagePainter|Coil|Glide|Picasso|painterResource" --type kotlin
```

```powershell
rg -ln "<vector|<bitmap|app:srcCompat|android:src=" --glob "**/res/drawable*/**"
rg -n "Image\(|AsyncImage|rememberAsyncImagePainter|Coil|Glide|Picasso|painterResource" --type kotlin
```

Record: illustration style (flat vector, photographic, mixed), empty-state/onboarding artwork
present or absent, image-loading library, raster vs vector mix for decorative imagery (UI glyphs are
Pass 6's job). One paragraph + a short list of the notable illustration assets is enough.

### Copy / strings census

```bash
rg -c "<string " --glob "**/res/values*/strings.xml"
rg -n "android:text=\"[^@]" --glob "**/res/layout*/**"
rg -n "Text\(\s*\"|stringResource\(" --type kotlin
```

```powershell
rg -c "<string " --glob "**/res/values*/strings.xml"
rg -n "android:text=\"[^@]" --glob "**/res/layout*/**"
rg -n "Text\(\s*\"|stringResource\(" --type kotlin
```

Record: total string count, how many locales exist (`values-<lang>/strings.xml`), and the volume of
**hardcoded** (non-`@string/`) UI text — hardcoded literals are both a localization risk and the
inputs prompt 06 needs to declare frozen vs redesignable strings (D13, the COPY-ticket pipeline). One
table (locale → string count) plus the hardcoded-literal count and worst-offender files.

## Output: "Current Design System" Doc Structure

Produce a single section (embedded in `docs/factory/audits/DESIGN_AUDIT.md` per
`09-templates/design-audit-report.md`, and referenced verbatim by the design handover) with exactly
these subsections:

1. **Theme overview** — theme parents per activity, toolkit split, Material library/Compose BOM
   versions, dark theme support status.
2. **Color census** — table from Pass 1.
3. **Typography census** — fonts + sp-size table from Pass 2.
4. **Spacing census** — dp table + inferred grid from Pass 3.
5. **Shape & elevation** — tables from Passes 4–5.
6. **Iconography** — census from Pass 6.
7. **Component inventory** — table from Pass 7 with keep/replace/absorb classification.
8. **Motion & interaction census** — from Pass 8: transition/animator/Lottie inventory, hardcoded
   duration list, interaction-state matrix, gesture map. This subsection is consumed verbatim by the
   design handover's Motion & Interaction Inventory section (`05-handover/DESIGN_HANDOVER_SCHEMA.md`)
   and feeds the motion target spec in `03-design/10-motion-design.md`.
9. **Imagery & copy census** — from Pass 9: imagery/illustration summary; strings table (locale →
   count) + hardcoded-literal count. Feeds the handover's Imagery & Illustration Direction and
   Localization & Content Constraints (frozen-strings) sections.
10. **Divergence report** — see below.

## Divergence Report

The punchline of the extraction. One summary table:

| Dimension | Distinct values | Tokenized | One-off / rogue | Divergence % | Worst offenders (files) |
|---|---|---|---|---|---|
| Color | 47 | 9 | 38 | 81% | `fragment_home.xml` (6 literals), `PaywallView.kt` (4) |
| Text size | 11 | 5 | 6 | 55% | … |
| Spacing | 19 | 3 | 16 | 84% | … |
| Corner radius | 6 | 1 | 5 | 83% | … |

Divergence % = one-off values ÷ distinct values. Interpretation bands for the audit narrative:
< 20% healthy (token layer exists and is followed); 20–50% drifting (enforce + backfill); > 50%
no effective design system (redesign must start at tokens — doctrine "tokens before screens",
`03-design/README.md`). Each row with divergence > 50% becomes one SYS-class DES finding in
`03-design/01-design-audit.md`.

## Quality Bar

- [ ] All 9 passes executed for both XML and Compose (or toolkit absence explicitly recorded).
- [ ] Every census table includes counts, not just lists.
- [ ] Pass 8 motion census produced: transition/Lottie inventory, hardcoded-duration list,
      interaction-state matrix, gesture map — ready for the handover's Motion & Interaction Inventory.
- [ ] Pass 9 imagery + strings census produced (strings table includes locale list and hardcoded
      count) for the handover's imagery and frozen-strings sections.
- [ ] Divergence report computed with worst-offender file paths.
- [ ] dp-based text sizes and missing `values-night` flagged to the accessibility and audit docs.
- [ ] Output embedded in `docs/factory/audits/DESIGN_AUDIT.md` and cross-linked from
      `docs/factory/PROJECT_STATE.md` (Design System Status field).
