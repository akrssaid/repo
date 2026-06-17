# Component Library Method

This document defines the method for building a target app's reusable UI component library during
redesign. It is consumed by `01-core/prompts/09-ui-implementation.md` (P6 — Redesign
Implementation) and by the design phases (`01-core/prompts/05-design-audit.md`,
`07-design-redesign.md`) when deciding what gets standardized. The library is how a redesign stays
consistent after the redesign agent leaves: screens stop styling themselves and start composing
shared, token-driven components. Inputs come from the token and pattern extraction in
`03-design/04-design-system-extraction.md`; Material 3 mapping rules live in
`03-design/05-material3-modernization.md`.

---

## 1. Component Audit (census of duplicated patterns)

Start from the pattern inventory produced by `03-design/04-design-system-extraction.md` (which
lists recurring visual patterns per screen). Convert it into a duplication census. **The census
counts DISTINCT implementations, not call sites** (rule 3 below) — a grep hit count is a call-site
count and is the wrong number. The procedure below makes "distinct" operational.

1. **Find candidate sites.** For each recurring pattern (button, card, list row, header, dialog,
   empty view, spinner, error view), list the sites where it is rendered. The grep below is a
   *site-finder*, not a counter — its `Measure-Object` total is the call-site count, which you
   will collapse in step 2:

   ```powershell
   # Compose: locate button render sites (one line per call site)
   Get-ChildItem -Recurse app\src -Filter *.kt | Select-String "Button\(" |
     Select-Object Path, LineNumber, Line
   ```

   ```bash
   # Compose: locate button render sites (one line per call site)
   grep -rn --include=*.kt "Button(" app/src
   ```

   ```powershell
   # XML: layouts defining their own button appearance
   Get-ChildItem -Recurse app\src\main\res\layout -Filter *.xml |
     Select-String "<Button|MaterialButton" | Select-Object Path, LineNumber, Line
   ```

   ```bash
   # XML: layouts defining their own button appearance
   grep -rnE "<Button|MaterialButton" app/src/main/res/layout
   ```

2. **Group by implementation signature, then count groups.** Take the site list and collapse it
   into distinct implementations by signature — two sites are the *same* implementation when they
   produce visually/structurally equivalent output. The signature for a:
   - **Compose pattern** = the called composable's fully-qualified name **plus** the set of
     styling args that change appearance (`shape`, `colors`, `contentPadding`, height/min-size
     modifiers, custom `Modifier.background`/`border`). Sites calling the same composable with
     equivalent styling args are **one** implementation; the same M3 `Button` styled three
     different ways inline is **three**. Sites calling an already-shared `AppButton` collapse to
     **one** regardless of how many screens call it.
   - **XML pattern** = the widget class **plus** the resolved `style`/applied appearance attrs.
     Distinct styles (or inline appearance overrides) = distinct implementations; many `<include>`s
     of one `view_*` layout = one.

   Do this by reading each site (not by counting grep lines): bucket sites whose signatures match,
   name each bucket, and the **number of buckets** is "Distinct implementations". Record screens
   touched per pattern from the union of the buckets' sites.

3. **Rule (definition of distinct).** "Distinct implementations" means visually or structurally
   different code paths, **not** call sites. Six screens calling one shared composable = **1**
   implementation (already healthy); one M3 primitive styled six different ways across six files =
   **6** implementations (the duplication the census exists to surface).

4. Fill the census table (it lands in the implementation handover per
   `05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md` §3 Component Specifications):

   | Pattern | Distinct implementations | Screens affected | Visual variance observed | Promote? |
   |---|---|---|---|---|
   | Primary button | 6 | 9 | 3 corner radii, 2 heights | Yes |
   | Empty state | 4 | 7 | different illustrations, copy tone | Yes |
   | Settings row | 1 | 1 | n/a | No |

## 2. Promotion Rules

Promote a pattern to a shared component when **either** condition holds:

1. **3+ usages rule**: the pattern appears (or will appear post-redesign) on 3 or more screens.
2. **Core-brand surface rule**: the pattern carries brand identity regardless of count — top app
   bar, primary button, splash-adjacent loading state, paywall card. One inconsistent instance of
   these is visible to every user.

Do NOT promote: single-screen layouts, speculative variants ("we might need a tertiary button"),
or thin wrappers that add zero tokens/behavior over the M3 component (call M3 directly instead).
Every promotion decision gets one row in the census table with the rule that triggered it.

---

## 3. Library Structure — Compose Apps

### 3.1 Package conventions

All shared components live in one package: `ui/components/` (single module apps) or a
`:core:designsystem` module (multi-module apps). Naming: `App` prefix, M3 noun, one file per
component family.

| Component | File | Wraps | Purpose |
|---|---|---|---|
| `AppButton` (+`AppOutlinedButton`, `AppTextButton`) | `AppButton.kt` | M3 `Button` family | Single source for shape, height, typography |
| `AppCard` | `AppCard.kt` | M3 `Card`/`ElevatedCard` | Corner radius, elevation, padding tokens |
| `AppTopBar` | `AppTopBar.kt` | M3 `TopAppBar` | Title style, nav icon, scroll behavior |
| `AppEmptyState` | `AppEmptyState.kt` | Column composition | Icon + title + body + optional action |
| `AppLoadingState` | `AppLoadingState.kt` | M3 indicator | Full-screen and inline variants |
| `AppErrorState` | `AppErrorState.kt` | Column composition | Message + retry slot |
| `AppOfflineState` | `AppOfflineState.kt` | Column composition | No-connection message + retry slot (the `offline` state from D12's per-screen states contract) |

The four state components — `AppEmptyState`, `AppLoadingState`, `AppErrorState`,
`AppOfflineState` — are the **canonical** rendering for the per-screen states contract in D12
(default/empty/loading/error/offline). Every data-loading screen renders those conditions only
through these (doctrine, Section 5); they are never re-improvised per screen.

Rules: components consume **theme tokens only** (`MaterialTheme.colorScheme/typography/shapes` as
configured by the app theme from `03-design/04-design-system-extraction.md`) — never hardcoded
colors/dp for brand-bearing values. Every component takes `modifier: Modifier = Modifier` as the
first optional parameter. Variation is expressed through **slot APIs** (composable lambdas), not
boolean explosion.

### 3.1a Per-component interaction-state requirement

Every promoted **interactive** component (button, card-as-button, list row, chip, toggle, top-bar
action — anything clickable/focusable) must carry an explicit **interaction-state spec**, and the
component must implement each applicable state. A component that does not define its states is an
incomplete deliverable, not a component. The mandatory states:

| State | Requirement |
|---|---|
| `pressed` | Visible press feedback — M3 state-layer (`pressed` ≈ 12% overlay) or an equivalent token-driven delta; never silent |
| `disabled` | Reduced-emphasis token treatment **and** the control is genuinely non-interactive (no `onClick` fires); enforce via `enabled` param, not opacity alone |
| `focused` | A visible focus indicator at ≥ 3:1 contrast for keyboard/D-pad/switch nav (ties to the keyboard pass in `03-design/09-accessibility-review.md` §2.6) |

Plus, where the component has them, `selected`/`error`/`loading` (a button with an in-progress
state shows a determinate/indeterminate indicator, not a frozen label). These per-component states
are exactly the **States** row required by the IMPLEMENTATION handover's §3 Component
Specifications (`05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md`): enabled, disabled, pressed,
focused, selected, error, loading — every state that applies, with the visual delta per state. The
component's spec in `COMPONENT_INVENTORY.md` (Section 6) records which states it implements so the
state spec travels with the component.

Screen-level states (default/empty/loading/error/offline) are a **different axis** and are owned
by the four state components above (Section 5) — do not conflate a screen's offline state with a
button's disabled state. Both axes must be specified.

### 3.2 Code skeleton (canonical example — AppEmptyState)

```kotlin
package com.example.app.ui.components

@Composable
fun AppEmptyState(
    title: String,
    modifier: Modifier = Modifier,
    body: String? = null,
    icon: @Composable (() -> Unit)? = null,   // slot: any visual, not just an ImageVector
    action: @Composable (() -> Unit)? = null, // slot: usually an AppButton
) {
    Column(
        modifier = modifier.fillMaxSize().padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center,
    ) {
        icon?.let { it(); Spacer(Modifier.height(16.dp)) }
        Text(title, style = MaterialTheme.typography.titleLarge, textAlign = TextAlign.Center)
        body?.let {
            Spacer(Modifier.height(8.dp))
            Text(it, style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant, textAlign = TextAlign.Center)
        }
        action?.let { Spacer(Modifier.height(24.dp)); it() }
    }
}

@Preview(name = "Empty - light", showBackground = true)
@Preview(name = "Empty - dark", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Preview(name = "Empty - 200% font", fontScale = 2.0f, showBackground = true)
@Composable
private fun AppEmptyStatePreview() {
    AppTheme {
        AppEmptyState(
            title = "No downloads yet",
            body = "Files you download will appear here.",
            icon = { Icon(Icons.Outlined.Download, contentDescription = null) },
            action = { AppButton(text = "Browse", onClick = {}) },
        )
    }
}
```

Every component MUST ship the three preview annotations shown (light, dark, 200% font scale) —
the font-scale preview is the cheap half of the survival test in
`03-design/09-accessibility-review.md`.

## 4. Library Structure — XML Apps

For View-based apps not being migrated to Compose in this cycle:

1. **Style inheritance.** Define an app style tree in `res/values/styles.xml`, parented to
   Material 3 styles, one style per promoted component:

   ```xml
   <style name="Widget.App.Button" parent="Widget.Material3.Button">
       <item name="cornerRadius">@dimen/corner_button</item>
       <item name="android:minHeight">@dimen/touch_target_min</item>
   </style>
   ```

   Apply globally via theme defaults (`materialButtonStyle`, `materialCardViewStyle`,
   `toolbarStyle`) so unstyled usages inherit correctly, then delete per-view style attributes.
2. **`<include>` layouts** for composite components: `layout/view_empty_state.xml`,
   `view_loading_state.xml`, `view_error_state.xml`, `view_offline_state.xml`, each with stable child IDs
   (`@+id/empty_title`, `@+id/empty_action`). Screens embed them with
   `<include layout="@layout/view_empty_state" android:visibility="gone"/>` and toggle
   visibility; a small binding helper class per component (e.g. `EmptyStateBinder`) sets text and
   click listeners so copy/behavior wiring is also centralized.
3. Never duplicate dimensions: all radii, spacings, and heights come from `res/values/dimens.xml`
   entries named after the token sheet in `03-design/04-design-system-extraction.md`.

---

## 5. State Components Doctrine

**Every screen that loads data must render its empty, loading, error, and offline conditions
exclusively through `AppEmptyState`, `AppLoadingState`, `AppErrorState`, and `AppOfflineState`**
(or their XML includes) — the five-state contract from D12 (default/empty/loading/error/offline).
No screen-local spinners, no inline "Something went wrong" Texts, no bespoke empty illustrations
without going through the shared component's slots.

Rationale: state surfaces are where redesigns rot first — they are rarely in design mocks, so
developers improvise, and six months later the app has five spinners. Centralizing them is the
single highest-leverage consistency mechanism in this method. Enforcement: the screen-by-screen
verification in `07-checklists/design-modernization.md` includes a row per screen for
Empty/Loading/Error/Offline = shared component Y/N.

## 6. Documentation Requirement

Maintain a component inventory at `docs/factory/reports/COMPONENT_INVENTORY.md`, updated whenever
a component is added or a screen adopts one:

| Component | Purpose | Slots / key params | States implemented | Screens used | Preview Y/N |
|---|---|---|---|---|---|
| AppButton | Primary CTA | text, onClick, enabled, leadingIcon slot | pressed, disabled, focused, loading | Home, Detail, Settings, Paywall | Y |
| AppEmptyState | Zero-data surface | title, body, icon slot, action slot | n/a (non-interactive container) | Downloads, Search, History | Y |

A component missing from this table does not exist as far as review is concerned; a table row
with Preview = N is an open defect (Effort S).

## 7. Drift Prevention

Lint-style review rule applied to every PR/change-set after the library lands (enforced during
`01-core/prompts/15-final-verification.md` and ongoing maintenance):

1. **No one-off styled components.** A new screen may not introduce a locally-styled button,
   card, top bar, or state view. Diff heuristic — flag any new file matching:

   ```powershell
   # New composables that style M3 primitives directly outside ui/components/
   Get-ChildItem -Recurse app\src -Filter *.kt | Where-Object FullName -NotMatch "ui\\components" |
     Select-String "ButtonDefaults\.|CardDefaults\.|RoundedCornerShape\(" 
   ```

   Hits require either (a) refactor to a shared component, or (b) a one-line written
   justification in the change description ("intentionally unique because …"). Silence is a
   review failure.
2. **No raw hex colors / magic dp** outside the theme and `ui/components/`: grep `Color(0x` and
   compare new `.dp` literals against the token sheet.
3. **Census refresh**: rerun the Section 1 census at each release prep; any pattern that crossed
   the 3-usage threshold since last release gets promoted before release.

### Acceptance checklist

- [ ] Census counts DISTINCT implementations (grouped by signature per §1 steps 2–3), NOT grep call-site totals
- [ ] Census table complete with promote/don't-promote decision per pattern
- [ ] The four canonical state components exist (Empty, Loading, Error, Offline) plus Button, Card, TopBar
- [ ] Every interactive component defines and implements pressed/disabled/focused (+ selected/error/loading where applicable) per §3.1a and the IMPL schema §3
- [ ] Components consume theme tokens only; no hardcoded brand values
- [ ] Every component has light/dark/200%-font previews (Compose) or a populated include sample (XML)
- [ ] All data-loading screens use shared state components for empty/loading/error/offline (doctrine, Section 5)
- [ ] `COMPONENT_INVENTORY.md` exists, matches the code, and records each component's implemented states
- [ ] Drift-prevention greps run clean or every hit is justified
