# Material 3 Essentials

Working reference for Material 3 (M3) facts agents need during design audits, redesigns, and
implementation (P5–P6): the color-role system, type scale, shape scale, elevation model, component
selection rules, the M2→M3 visual diff, and a misuse table for review work. Token values below are
the M3 baseline spec; component specifics evolve (M3 Expressive is rolling out through 2025–2026)
— **verify against current official docs** (m3.material.io and the Compose M3 release notes) when
a value is load-bearing. Procedures for *applying* M3 live in
`03-design/05-material3-modernization.md`.

**last reviewed: 2026-06-10**

---

## Color System

M3 colors are **roles**, not hex values. A scheme is generated (from a seed or dynamic source) and
every component consumes roles, so themes, dark mode, and dynamic color "just work" — if and only
if the app never hardcodes hex.

### The 5 key role families

Each family provides four slots: the role, `on<Role>` (content placed on it), `<role>Container`
(a softer fill), and `on<Role>Container`.

| Family | Use for | Rules |
|---|---|---|
| **Primary** | The app's signature actions: FAB, filled buttons, active nav indicator, key toggles ON state | One visual "voice" per screen — if everything is primary, nothing is |
| **Secondary** | Supporting UI: filter chips, tonal buttons, less prominent accents | Must read as "related to but quieter than primary"; not a second brand color free-for-all |
| **Tertiary** | Deliberate contrast accents: input field decoration, highlights, promotional moments | Sparingly — see misuse table; it exists for *balance*, not extra emphasis |
| **Error** | Error states only: invalid fields, destructive confirmation, failure banners | Never decorative; never used for "important" non-error content |
| **Neutral (surface/outline)** | Backgrounds, cards, sheets, dividers, disabled states | Carries the surface hierarchy: `surface`, `surfaceContainer{Lowest…Highest}`, `outline`, `outlineVariant` |

Pairing rule: content on `X` uses `onX`, content on `xContainer` uses `onXContainer` — these pairs
are contrast-guaranteed by the scheme generator. Mixing pairs (e.g. `onPrimary` text on
`primaryContainer`) is the most common accessibility defect found in audits.

### Surface tint and the container ladder

- Surfaces gain hierarchy via **tonal color steps**, not shadows: `surfaceContainerLowest` →
  `surfaceContainerLow` → `surfaceContainer` → `surfaceContainerHigh` → `surfaceContainerHighest`.
  Higher = more prominent (dialogs, menus sit high; page background sits at `surface`).
- Legacy Compose M3 applied a primary-tinted elevation overlay (`surfaceTint` + `tonalElevation`);
  the container ladder is the current idiom — prefer explicit container roles over tonal-elevation
  arithmetic in new code.

### Dynamic color and the brand-seed fallback

- On Android 12+ (API 31+), dynamic color derives the entire scheme from the user's wallpaper:
  `dynamicLightColorScheme(context)` / `dynamicDarkColorScheme(context)`.
- Every app must also define a **brand seed** scheme (generated from one brand hex via the
  Material Theme Builder or `material-kolor`-style generation) used when: API < 31, dynamic color
  is disabled, or brand consistency is required (e.g. store screenshots —
  `04-aso/workflows/screenshot-optimization.md` shoots with the brand scheme for consistency).
- Decision: utility apps default to **dynamic color ON with brand-seed fallback**; opt out only
  when color is itself the brand (rare for this portfolio).

---

## Typography — the 15-style type scale

Baseline values (default Roboto metrics; line heights rounded). Sizes in sp.

| Style | Size / Weight | Use |
|---|---|---|
| displayLarge | 57 / Regular | Hero numerals, splash-level statements; rare in utility apps |
| displayMedium | 45 / Regular | Large stat readouts (e.g. "2.4 GB recovered") |
| displaySmall | 36 / Regular | Section hero text |
| headlineLarge | 32 / Regular | Top-of-page headings on large screens |
| headlineMedium | 28 / Regular | Screen titles outside app bars, empty-state headlines |
| headlineSmall | 24 / Regular | Dialog-scale headings, card group headers |
| titleLarge | 22 / Regular | App bar titles, dialog titles |
| titleMedium | 16 / Medium | List item primary text (emphasized), card titles |
| titleSmall | 14 / Medium | Dense list titles, tab labels |
| bodyLarge | 16 / Regular | Primary reading text, list item text |
| bodyMedium | 14 / Regular | Default body, supporting list text |
| bodySmall | 12 / Regular | Captions, metadata, timestamps |
| labelLarge | 14 / Medium | Button text, prominent chips |
| labelMedium | 12 / Medium | Small buttons, nav bar labels |
| labelSmall | 11 / Medium | Overlines, dense badges |

Rules: pick **3–5 styles per screen** maximum; never set raw `fontSize` in screens — only theme
styles (this is what makes user font-scaling and later rebrands cheap); body text below
`bodyMedium` (14sp) is an accessibility flag in `03-design/09-accessibility-review.md`.

---

## Shape Scale

| Token | Corner radius | Typical components |
|---|---|---|
| extraSmall | 4dp | Text fields (filled), snackbars, small menus |
| small | 8dp | Chips, small cards, tooltips |
| medium | 12dp | Cards, list-item containers, smaller dialogs |
| large | 16dp | FABs, navigation drawers, large cards |
| extraLarge | 28dp | Dialogs, bottom sheets (top corners), search bars |
| full | pill/circle | Buttons, badges, sliders, icon buttons |

Doctrine: customize at most one or two tokens to express brand (e.g. squarer = technical, rounder
= friendly) and let every component inherit — never set per-component radii ad hoc. M3 Expressive
adds shape morphing/variety on top of this scale; treat as enhancement, not baseline.

---

## Elevation: tonal vs shadow

| | Tonal (color) | Shadow |
|---|---|---|
| What it is | Higher surfaces use higher `surfaceContainer` steps (or tint overlay) | Classic drop shadow |
| Dark theme | Primary mechanism — shadows are nearly invisible on dark | Weak; never the only cue |
| M3 default | Most components express elevation tonally | Retained subtly on FABs, menus, dragged items |

Levels 0–5 map to 0/1/3/6/8/12dp shadow equivalents. Rule: in dark theme, hierarchy must survive
with shadows off — if two stacked surfaces look identical, the container roles are wrong, not the
shadow values.

---

## Motion

The factory's motion vocabulary. This section is the single source the DESIGN handover's **Motion &
Interaction Inventory** quotes and the method doc `03-design/10-motion-design.md` applies; the
IMPLEMENTATION handover's §Motion Specification and per-ticket `Motion:` fields are written against
the terms defined here (the end-to-end pipeline is D14). Numbers below are the M3 baseline ranges —
**approximate, verify against current official docs** (m3.material.io motion + the Compose
`MotionScheme`/`AnimationSpec` APIs) when a duration or curve is load-bearing.

### Duration scale (approximate bands)

Pick a duration by the **distance and area** the element travels/changes, not by taste. Larger or
full-screen transitions sit at the long end; small, in-place state changes at the short end.

| Band | Approx. range | Use for |
|---|---|---|
| **Short** | ~50–200 ms | Small, in-place changes: ripples, selection/toggle state, icon swaps, hover/press feedback, small fades |
| **Medium** | ~250–400 ms | Component-level motion: expanding cards, chips, menus, bottom-sheet partial expand, snackbar enter/exit |
| **Long** | ~450–600 ms | Large/full-screen transitions: container transform, page/shared-axis transitions, large incoming surfaces |

Rules: enter (incoming) motion may run slightly longer than the matching exit; exits should feel
quicker so the UI never feels sluggish to leave. Never exceed the long band for routine navigation —
motion past ~600 ms reads as lag, not polish.

### Easing — standard vs emphasized

M3 expresses speed *over time* with two easing families. Choosing the wrong one is the most common
motion defect after wrong duration.

| Family | Feel | When |
|---|---|---|
| **Standard** | Gentle accelerate-then-decelerate; symmetric-ish | Default for most transitions, small movements, elements that begin and end on screen |
| **Emphasized** | Slow start, fast middle, soft settle (more dramatic deceleration) | The hero/primary transition of a flow — full-screen and container transforms, the motion you *want* noticed |

Each family has **accelerate** (for elements *leaving* the screen — quick exit) and **decelerate**
(for elements *entering* — soft arrival) variants; use accelerate on exits, decelerate on enters,
and the full curve for on-screen-to-on-screen movement. Linear easing is reserved for continuous
indicators (progress, loaders), never for spatial transitions.

### Transition pattern decision table

The three workhorse M3 navigation transitions and when each applies. Choose by the **relationship
between the two screens/elements**, not by what looks nice:

| Pattern | Use when | Typical duration / easing | Example |
|---|---|---|---|
| **Container transform** | A persistent element *becomes* the next surface — there is a clear visual continuity (a card, list row, FAB, or search bar expands into a detail screen or sheet) | Long band, **emphasized** | List item → detail screen; FAB → create sheet; search bar → search results |
| **Shared axis** | Two UI elements have a **spatial or sequential** relationship — forward/back in a flow, sibling tabs, stepper pages. Direction encodes meaning (X = lateral peers, Y = vertical hierarchy, Z = depth/drill-in) | Medium–long, standard or emphasized | Onboarding step 2→3 (X); parent list → child list (Z); date-picker month change (X) |
| **Fade through** | The two states are **unrelated** — no spatial continuity, just a content swap (switching bottom-nav destinations, refreshing a whole panel) | Medium, standard (outgoing fades out, incoming fades in after, no overlap of full opacity) | Bottom-nav tab A → tab B; empty-state → loaded content |

Default selector: continuity exists → container transform; a direction/sequence exists → shared
axis on that axis; neither → fade through. Bottom-nav destination changes are fade-through, *not*
shared-axis (the destinations are peers with no inherent order).

### Elevation on scroll

App bars and other surfaces change emphasis as content scrolls under them: at rest the top app bar
sits at `surface` (flush, no separation); once content scrolls beneath it, it shifts to a higher
`surfaceContainer` step (and/or gains a subtle shadow) to signal that content is now passing under
it. In Compose this is driven by `TopAppBarScrollBehavior` + the scroll connection — the color/
elevation change is a short-band transition tied to scroll offset, not a one-shot animation. Use it
to communicate layering; do not animate the app-bar color on every frame of every scroll pixel
manually.

### Reduced-motion fallback (mandatory)

When the system "Remove animations" / reduced-motion setting is on (or the platform reports a
reduced animator scale), **large spatial transitions must degrade to a simple cross-fade or an
instant cut** — never play container transforms or long shared-axis moves. Functional feedback
(press states, progress) stays. Every motion spec in the IMPLEMENTATION handover carries an explicit
**reduced-motion fallback** column (D14); a redesign that specifies motion without its fallback is
incomplete. Respecting reduced motion is an accessibility requirement, checked in
`03-design/09-accessibility-review.md`.

---

## Component Selection Guide

| Decision | Rule |
|---|---|
| Bottom nav bar vs nav rail vs drawer | 3–5 top destinations on phones → **navigation bar**; same app on ≥600dp width → **rail** (mandatory thinking at targetSdk 36 adaptive-apps rules, see `08-knowledge/android/version-matrix.md`); 6+ destinations or rarely-switched sections → **modal drawer** — but 6+ usually means the IA is wrong |
| FAB vs prominent filled button | FAB = the screen's *one* constructive, repeated action (compose, add, scan) floating over scrollable content. Inline flows, forms, confirmations → filled button. Never both competing on one screen; never a FAB for "next" |
| Dialog vs bottom sheet | Dialog = interrupt requiring a decision (confirm delete, simple pick, error needing acknowledgment). Bottom sheet = optional tools, share targets, multi-option pickers, anything benefiting from reachability/partial expansion. Long content in a dialog → it should be a sheet or screen |
| Snackbar vs dialog for feedback | Result-of-action feedback with optional undo → snackbar. Blocking problem the user must resolve → dialog. Success of an explicit user action rarely needs a dialog |
| Chips vs buttons | Chips filter/refine/represent entities in context; buttons commit actions. A chip that navigates is a misuse |

---

## M2 → M3: what users actually notice

| Aspect | M2 | M3 |
|---|---|---|
| Corners | Mostly 4dp, sharp feel | Rounder everywhere (12–28dp, pill buttons) |
| Surfaces | White/gray + shadow elevation | Tonal surfaces — colored hierarchy, softer depth |
| Color | Primary/secondary, fixed brand | Role system + dynamic (wallpaper) color |
| Type | Smaller, denser defaults | Larger, more spacious scale |
| App bar | Colored primary app bar | Surface-colored, larger title variants, scroll behaviors |
| Buttons | Sharp-corner contained/outlined | Pill filled/tonal/elevated/outlined/text hierarchy |
| Overall read | "An app from 2018" | "Belongs on a current Pixel" |

For audits: an M2 app is not "broken", but it visibly dates the product — the design audit
(`03-design/01-design-audit.md`) scores this as modernization debt, severity per app category
competitiveness.

---

## Common M3 Misuse Table

| Misuse | Why it is wrong | Correct move |
|---|---|---|
| Tertiary as accent-spam (sprinkled on random UI for "pop") | Destroys the emphasis hierarchy; tertiary is a balancing voice | Re-emphasize with primary/primaryContainer; reserve tertiary for one deliberate accent context |
| Mismatched on-colors (e.g. `onSurface` text over `primaryContainer`) | Breaks generated contrast guarantees; fails WCAG in some schemes | Always use the paired `on*` role of the actual background |
| M2 elevation overlays / gray-tint hacks in dark theme | Double-tints surfaces once container roles are used; muddy dark UI | Use `surfaceContainer*` steps; delete manual overlay logic |
| Hardcoded hex matching the light scheme | Dark theme + dynamic color silently break | Roles only; hex lives solely in the seed/theme definition |
| Primary-colored top app bar (M2 habit) | Fights M3 surface hierarchy; clashes under dynamic color | Surface app bar; express brand via primary on actions/indicators |
| `error` role for warnings/emphasis | Cries wolf; real errors lose meaning | Warnings use tertiary/custom extended roles; error = errors only |
| Disabling dynamic color "for brand" without a generated brand scheme | Ships one hand-picked palette that fails contrast in dark theme | Generate the full scheme from a seed; then decide dynamic on/off |

Cross-references: extraction of an app's current tokens (incl. the motion/interaction census,
Extraction Pass 8) → `03-design/04-design-system-extraction.md`; applying this doc to a redesign →
`03-design/05-material3-modernization.md`; applying the Motion section above as a procedure →
`03-design/10-motion-design.md`; copy/voice doctrine → `08-knowledge/design/ux-writing-essentials.md`;
UX-level doctrine → `08-knowledge/design/mobile-ux-principles.md`.
