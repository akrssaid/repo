# Adaptive Layout Method

This document defines the method for making a redesigned target app adaptive across phones,
tablets, and foldables: how to classify the window, which canonical layout to choose, how to
decide navigation and pane behavior per size class, how to handle foldable postures, and how
adaptive specs flow through the design→engineering handover. It is consumed by the design phases
(`01-core/prompts/07-design-redesign.md`) when proposing adaptive behavior and by
`01-core/prompts/09-ui-implementation.md` (P6 — Redesign Implementation) when building it.

Conventions (IDs, scales, canonical paths) live in `01-core/CONVENTIONS.md` and are cited, not
restated. Material 3 component and token rules live in `03-design/05-material3-modernization.md`;
this file is the operational adaptive procedure.

**Scope gate first — read Section 7 before anything else.** If the app is declared phone-only,
adaptive work is OUT of scope and you stop after recording that declaration; do not produce
tablet specs or demand tablet screenshots.

---

## 1. Window Size Classes

Adaptive layout keys off **window** size (the area the app actually occupies), never physical
device type — a phone in landscape, a tablet in a split-screen third, and a foldable mid-fold are
all distinct windows. Use the Material 3 / Jetpack `WindowSizeClass` breakpoints:

| Class | Width breakpoint | Height breakpoint | Typical window |
|---|---|---|---|
| Compact | width < 600 dp | height < 480 dp | Phone portrait; small split-screen |
| Medium | 600 dp ≤ width < 840 dp | 480 dp ≤ height < 900 dp | Phone landscape; small tablet portrait; unfolded inner display (portrait) |
| Expanded | width ≥ 840 dp | height ≥ 900 dp | Tablet landscape; large tablet portrait; desktop window |

Width class drives most layout decisions; height class matters for vertical affordances (bottom
sheets, full-height panes, whether a `TopAppBar` should be `Large`/collapsing). Classify both, and
**re-classify on every configuration change** — rotation, fold/unfold, and multi-window resize all
change the window. Never branch on `Build.MODEL`, screen inches, or `smallestScreenWidthDp`
resource qualifiers as the primary signal; classify the live window.

---

## 2. Canonical Layouts

Three canonical layouts cover the large majority of app screens. Pick per screen, per size class.

| Layout | What it is | Use when | Compact → Expanded behavior |
|---|---|---|---|
| **List-Detail** | A list pane and a detail pane | The screen is "browse a collection, open one item" (mail, notes, settings, files) | Compact: list fills, detail is a separate destination (navigate). Expanded: list + detail side by side; selecting a row updates the detail pane in place |
| **Feed** | A scrolling grid/list of equal items | Home feeds, galleries, product grids, dashboards | Compact: 1 column. Medium: 2 columns. Expanded: 3+ columns, capped by a max content width so cards don't stretch |
| **Supporting Pane** | A primary content area plus a secondary supporting pane (filters, metadata, related, tools) | The screen has primary content with always-relevant secondary info | Compact: supporting content collapses into a bottom sheet / expandable section / separate route. Expanded: supporting pane docked beside primary |

Rules that apply to all three:

- **Cap content width.** Never let text columns or single content blocks stretch full-width on
  expanded windows — pin a max width (M3 guidance ~840 dp for body content) and center, or move to
  a multi-pane layout. Full-bleed line lengths are an expanded-window readability finding.
- **Two-pane state survives rotation/fold.** When a detail pane is open and the window shrinks to
  compact, the selected item must stay selected (navigate to its detail route), and growing back
  to expanded must restore the pane — no lost selection.
- **One source of truth for selection.** The selected item lives in the ViewModel/nav state, not
  in pane-local state, so both panes and the compact route read the same value.

---

## 3. Per-Screen-Class Decision Table

For each screen, fill this decision table (it feeds the proposal's per-screen adaptive behavior,
Section 6). The defaults below are the factory's canonical mapping; deviations need a one-line
rationale.

| Aspect | Compact | Medium | Expanded |
|---|---|---|---|
| Navigation | Bottom **nav bar** (3–5 top destinations) | **Nav rail** (left edge) | **Nav rail**, or **permanent/modal drawer** when destinations > 5 or grouped |
| Panes | Single pane | List-detail may go two-pane if width ≥ ~720 dp | Two-pane (list-detail / supporting pane) |
| FAB | Bottom-end, above nav bar | Anchored to the nav rail's top, or bottom-end of content pane | In the content pane (often list pane top), not floating over both panes |
| Top app bar | Small / center | Small | `Large`/medium collapsing where height allows |
| Content width | Full width | Full width or 2-col | Capped + centered, or multi-pane |

Navigation progression is **nav bar → nav rail → drawer** as width grows — never keep a bottom nav
bar on an expanded window (it wastes the horizontal axis and reads as a blown-up phone). The same
destination set must be reachable in every form; do not hide destinations on smaller classes.

---

## 4. Foldable Postures

Foldables introduce **postures** (physical hinge state) on top of size classes. Read posture from
Jetpack WindowManager `WindowInfoTracker` → `FoldingFeature`; do not infer it from size alone.

| Posture | Hinge state | Design response |
|---|---|---|
| **Tabletop** | Half-opened, hinge horizontal (laptop-like) | Split the UI across the fold: content/media in the top half, controls/list in the bottom half (e.g. video above, comments below; map above, list below) |
| **Book** | Half-opened, hinge vertical (book-like) | Treat as a natural two-pane boundary: list-detail or page-spread with the hinge as the gutter |
| Flat (unfolded) | Fully open | Classify by size class (usually Expanded inner display) and use the Section 2/3 layout |

**Hinge / fold avoidance is mandatory.** When a `FoldingFeature` is present and occluding
(`isSeparating` / a hinge with non-zero bounds):

- No interactive control, critical text, or media focal point may sit under the hinge bounds.
- Lay out so the fold falls **between** panes or sections (it becomes the gutter), never through
  the middle of a single content block or a tappable target.
- Apply hinge bounds as padding/inset to the two regions; reserve the hinge area as dead space.

If WindowManager is not a dependency and the app has no foldable users in analytics, foldable
posture handling may be deferred — but state that as an explicit decision in the handover's
Adaptive section, not by silence (mirror the Section 7 declaration discipline).

---

## 5. Landscape Rules for High-Traffic Screens

Even phone-only apps render in landscape (rotation, foldable cover displays, some launchers).
For every **traffic-High** screen (top-5 by traffic from the screen inventory,
`03-design/02-screen-inventory.md`):

- Landscape must be **usable**, not just non-crashing: no critical action below the fold with no
  scroll, no fixed-height container clipping content, no element overlapping the system bars.
- Convert vertical stacks of primary content to a row/two-pane where it reads better (a form with
  a hero image: image left, fields right in landscape).
- Keep the FAB and primary CTA reachable without scrolling.
- If the app locks orientation, that is a deliberate decision recorded in the handover — an app
  that locks portrait to avoid landscape work is flagged, not silently accepted.

---

## 6. How Adaptive Specs Flow Through the Handover

Adaptive behavior travels the same design→engineering pipeline as motion (D14 pattern), in three
stops:

1. **DESIGN handover → §Adaptive & Device Context** (`05-handover/DESIGN_HANDOVER_SCHEMA.md`, the
   section added in D15). Records: target device classes in scope, per-screen canonical layout
   choice, foldable support decision, and the phone-only declaration if applicable. This is the
   contract the proposal and IMPL build against.
2. **REDESIGN proposal → per-screen adaptive behavior.** For each screen, the proposal states the
   Section 3 decision-table row (nav treatment, pane behavior, FAB placement, content-width cap)
   and any posture handling — the design-side intent, screen by screen.
3. **IMPLEMENTATION handover → §Adaptive Layout Spec** (present in the IMPL schema
   `05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md` **when tablet/adaptive is in scope**, per D15).
   Each adaptive ticket carries:
   - the size-class → layout mapping as an exact spec (not "make it responsive"),
   - **acceptance criteria that name a tablet AVD** and both orientations (e.g. "on the
     `factory-tablet` AVD, expanded list-detail shows both panes; selecting a row updates detail
     in place; rotating to portrait keeps the selection"),
   - dependencies on the theme/component tickets.

Every proposal adaptive line maps to an IMPL ticket or is explicitly rejected — the same
no-silent-drop rule the motion pipeline uses.

---

## 7. When Adaptive Is OUT of Scope (phone-only apps)

Adaptive/tablet/foldable work is **opt-in per app**. Declare a screen set or the whole app
phone-only when ALL hold, and record the declaration in the DESIGN handover's §Adaptive & Device
Context:

- The app targets phones only (no tablet/foldable install base of note in Play Console device
  data), AND
- The product has no tablet/foldable roadmap this cycle, AND
- The team accepts the "stretched phone UI" appearance on large windows.

When phone-only is declared:

- This method stops at the declaration; no per-screen canonical layouts, no two-pane work, no
  posture handling.
- The IMPL schema's §Adaptive Layout Spec is omitted (it is in-scope-only per D15).
- **Prompt 11 must not demand tablet screenshots.** The screenshot-generation and capture steps
  read the phone-only declaration and capture phone form factors only; the absence of tablet
  store assets is then correct, not a gap. Make the declaration explicit so downstream prompts do
  not flag missing tablet captures.

Phone-only does **not** waive Section 5 — landscape rules still apply to High-traffic screens,
because phones rotate.

---

## 8. Compose Implementation Pointers

| Need | API |
|---|---|
| Classify the window | `currentWindowAdaptiveInfo()` (androidx.compose.material3.adaptive) or `calculateWindowSizeClass(activity)` (androidx.compose.material3.windowsizeclass) |
| Branch on class | `when (windowSizeClass.windowWidthSizeClass) { Compact / Medium / Expanded }` |
| List-detail scaffold | `NavigableListDetailPaneScaffold` / `ListDetailPaneScaffold` (androidx.compose.material3.adaptive.layout) — handles pane show/hide and back behavior |
| Supporting pane | `SupportingPaneScaffold` |
| Adaptive navigation | `NavigationSuiteScaffold` (androidx.compose.material3.adaptive.navigationsuite) — auto-swaps nav bar ↔ rail ↔ drawer by size class |
| Fold / posture | `WindowInfoTracker.windowLayoutInfo(...)` → `FoldingFeature` (orientation, occlusionType, bounds) |

Dependency note: the adaptive APIs live in the `androidx.compose.material3.adaptive:*` artifacts;
add them via `gradle/libs.versions.toml` and verify the latest stable version. Prefer the
canonical scaffolds over hand-rolled `if (isTablet)` branching — they encode the back/selection/
fold behavior this method requires and keep the two-pane state contract (Section 2) for free.

---

## 9. Capture Additions (tablet AVD)

When adaptive is in scope, the redesigned-screenshot set gains a tablet pass on top of the phone
pass defined in `03-design/03-screenshot-capture.md` (that file owns demo mode, the 10:00 clock,
naming `SCR-<id>-<state>-<theme>.png`, and directory layout — this is an addition, not a
replacement).

1. Create a tablet AVD (expanded window) and boot it:

   ```bash
   sdkmanager "system-images;android-36;google_apis_playstore;x86_64"
   avdmanager create avd -n factory-tablet -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_tablet
   emulator -avd factory-tablet -no-snapshot-load
   adb wait-for-device
   ```

   ```powershell
   sdkmanager "system-images;android-36;google_apis_playstore;x86_64"
   avdmanager create avd -n factory-tablet -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_tablet
   emulator -avd factory-tablet -no-snapshot-load
   adb wait-for-device
   ```

2. Capture **both orientations** for every adaptive screen (a tablet's two-pane layout is its
   point — capture it):

   ```bash
   adb shell settings put system accelerometer_rotation 0
   adb shell settings put system user_rotation 0    # portrait
   adb shell settings put system user_rotation 1    # landscape
   ```

   ```powershell
   adb shell settings put system accelerometer_rotation 0
   adb shell settings put system user_rotation 0    # portrait
   adb shell settings put system user_rotation 1    # landscape
   ```

3. Name captures with an orientation/device modifier in the optional `<modifier>` slot of the
   `03-design/03-screenshot-capture.md` naming convention (e.g. `SCR-003-default-light-tabland`,
   `SCR-003-default-light-tabport`) and run them through the same demo-mode and PNG-validation
   steps. These tablet captures are the evidence the IMPL §Adaptive Layout Spec acceptance criteria
   are checked against.

---

## Acceptance Checklist

- [ ] Scope decided: phone-only declared (Section 7) **or** adaptive in scope with target classes named in the DESIGN handover §Adaptive & Device Context
- [ ] Every in-scope screen has a filled Section 3 decision table (nav, panes, FAB, content width per class)
- [ ] Canonical layout chosen per screen (list-detail / feed / supporting pane) with two-pane state surviving rotation/fold
- [ ] Navigation progresses nav bar → nav rail → drawer by width; no bottom nav bar on expanded windows
- [ ] Foldable postures handled or explicitly deferred in the handover; nothing critical under the hinge
- [ ] Landscape verified usable on all traffic-High screens (Section 5), phone-only included
- [ ] Adaptive specs flow recorded: DESIGN §Adaptive & Device Context → proposal per-screen behavior → IMPL §Adaptive Layout Spec with tablet-AVD acceptance criteria
- [ ] If in scope: tablet AVD captures taken in both orientations per Section 9 and named per `03-design/03-screenshot-capture.md`
- [ ] If phone-only: declaration explicit so prompt 11 does not demand tablet screenshots
