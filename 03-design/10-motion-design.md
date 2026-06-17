# Motion Design Method

This document defines how the factory designs, specifies, and verifies UI motion for a target
Android app (D14, new in v1.1). Motion is a first-class design concern (`03-design/README.md`
doctrine): every transition and animation is *designed* — assigned a duration token, an easing
curve, and a reduced-motion fallback — never left to framework defaults. This method is consumed in
P5 by `01-core/prompts/07-design-redesign.md` (to author per-screen Motion lines) and
`08-implementation-handover.md` (to populate the IMPL schema's Motion Specification), and in P6 by
`09-ui-implementation.md` (to build the motion).

The current motion reality is censused at extraction (Pass 8 of
`03-design/04-design-system-extraction.md`); this doc supplies the *target* vocabulary and decision
rules. IDs, scales, and paths here are the canonical ones from `01-core/CONVENTIONS.md`.

## M3 motion tokens

These tokens are also defined in the motion section of
`08-knowledge/design/material3-essentials.md` — that file is the single source for the numeric
values; cite it, do not fork the numbers. Summarized here so the method is self-contained:

### Duration scale (~50–600ms)

| Band | Range | Typical use |
|---|---|---|
| Short | ~50–200 ms | Small-area, in-place changes: ripples, selection toggles, icon morphs, small fades |
| Medium | ~200–400 ms | Local component motion: expanding cards, chips, in-screen reveals, list item enter |
| Long | ~400–600 ms | Large-area / full-screen: navigation transitions, container transforms, bottom sheets |

Rule of thumb: the larger the area that moves and the farther it travels, the longer the duration.
Never exceed ~600 ms for routine UI motion — it reads as sluggish.

### Easing

| Easing | When | Curve intent |
|---|---|---|
| Standard | The default for most transitions: elements entering, exiting, or moving within the screen | Asymmetric ease-in-out; quick acceleration, gentle settle |
| Emphasized | Hero / high-attention moments: primary navigation transitions, container transforms | Stronger acceleration + a longer, more expressive deceleration |
| Linear | Only for continuous/looping motion (progress, indeterminate spinners) | Constant velocity |

Decelerate-only (ease-out) for elements **entering** the screen; accelerate-only (ease-in) for
elements **exiting**. Pair every duration token with an explicit easing — a duration without an
easing curve is an incomplete spec and fails the acceptance bar below.

## Transition pattern decision table

Choose the navigation transition by the **relationship** between the source and destination, not by
preference. (Material motion classes: `MaterialContainerTransform`, `MaterialSharedAxis`,
`MaterialFadeThrough`.)

| Navigation relationship | Pattern | Why | Duration band |
|---|---|---|---|
| Parent ↔ child (drill in/out): list item → detail, card → expanded view | **Container transform** | The tapped element visually becomes the destination; preserves object continuity | Long |
| Peer ↔ peer, same hierarchy level: tab → tab, step → step in a wizard, pager pages | **Shared axis** (X for lateral/next-prev, Y for up/down, Z for in/out depth) | A directional spatial model tells the user which way they moved | Medium–Long |
| Unrelated / no spatial relationship: switching bottom-nav top-level destinations, refreshing content | **Fade through** | No spatial metaphor fits; outgoing fades out, incoming fades in with a slight scale | Medium |
| In-place state change (same element, new content): count badge, selected state, inline expand | `animateContentSize` / `AnimatedContent` / `Crossfade` | Local change, not navigation | Short–Medium |

When the relationship is genuinely ambiguous, prefer **fade through** (the safest, least-assertive
option) over inventing a spatial metaphor that misleads.

## Predictive-back transition requirements

Predictive back (Android 14+, system-wide by Android 15) shows a live preview of the destination as
the user drags the back gesture. Requirements for every screen that can be backed out of:

1. **Opt in** — declare `android:enableOnBackInvokedCallback="true"` in the manifest `<application>`;
   migrate any legacy `onBackPressed()` override to `OnBackPressedDispatcher` /
   `OnBackInvokedCallback`. Compose: use `PredictiveBackHandler` (not the old `BackHandler`) where a
   custom animated back is needed.
2. **Animate the preview** — the back-progress transition must mirror the forward transition's
   pattern in reverse (a container-transform entry returns via a container-transform back; a
   shared-axis-X forward returns shared-axis-X reverse). A jump-cut back where the forward animated
   is a defect.
3. **Cancellable** — the gesture can be released mid-drag; the animation must reverse cleanly to the
   origin with no flicker or state loss.
4. **Cross-activity** transitions honor the system's back animation; do not suppress it with
   `overridePendingTransition(0, 0)`.

Every screen's Motion line (below) states its predictive-back behavior, or explicitly notes "system
default" when no custom animation is warranted.

## List & stagger animation rules

- **Item placement** — reordering, insertion, and removal in lists animate via
  `Modifier.animateItemPlacement()` (Compose) or `DiffUtil` + `RecyclerView` item animator (Views).
  Items must not jump.
- **Stagger** — entrance stagger (each item delayed slightly after the previous) is allowed only on
  first appearance of a list, capped so the **last** item is fully on screen within ~Long duration
  total. Never stagger on every scroll or data refresh — it makes the app feel slow.
- **No layout thrash** — reserve space for async content (skeletons) so loaded content does not shove
  the list (ties to audit H7, `03-design/01-design-audit.md`).
- Stagger and entrance animations are among the **first** things disabled under reduced motion (see
  contract below) — they degrade to an immediate, un-staggered appearance.

## Reduced-motion contract

The system "Remove animations" accessibility setting sets
`Settings.Global.ANIMATOR_DURATION_SCALE == 0`. **Every** animation must degrade gracefully when it
is 0: the end state appears immediately (or via an instant ≤ ~50 ms crossfade), with no movement,
parallax, scale, or stagger. The user must never lose information or an affordance because motion was
their only cue — motion is decoration on top of a state change that also works statically.

Detect it and branch:

```kotlin
// Views / general
val scale = Settings.Global.getFloat(
    contentResolver, Settings.Global.ANIMATOR_DURATION_SCALE, 1f
)
val reducedMotion = scale == 0f
```

```kotlin
// Compose
val context = LocalContext.current
val reducedMotion = remember {
    Settings.Global.getFloat(
        context.contentResolver, Settings.Global.ANIMATOR_DURATION_SCALE, 1f
    ) == 0f
}
// gate transitions: if (reducedMotion) Snapshot/instant else animate
```

Note that Android scales animator durations automatically for many framework animations, so a value
of 0 already zeroes much standard animation — but custom Compose animations, manual `ValueAnimator`s,
and Lottie do **not** read it automatically and must branch explicitly.

Test reduced motion on the emulator (both shells — same adb command):

```bash
adb shell settings put global animator_duration_scale 0   # simulate reduced motion
# ... exercise every animated flow ...
adb shell settings put global animator_duration_scale 1   # restore
```

```powershell
adb shell settings put global animator_duration_scale 0   # simulate reduced motion
# ... exercise every animated flow ...
adb shell settings put global animator_duration_scale 1   # restore
```

A flow that breaks, loses state, or hides an affordance at scale 0 is a defect, verified in
`03-design/12-design-qa.md` and gated by `07-checklists/accessibility.md`.

## Compose implementation pointers

| Need | API |
|---|---|
| Swap content (new screen/state) with a transition | `AnimatedContent` (set `transitionSpec`); `Crossfade` for a plain fade-through |
| Animate a size change in place | `Modifier.animateContentSize()` |
| Animate list insert/remove/reorder | `Modifier.animateItemPlacement()` (in `LazyColumn`/`LazyRow` item scope) |
| Show/hide an element | `AnimatedVisibility` with explicit `enter`/`exit` specs |
| Single-value tween/spring | `animate*AsState` (`animateDpAsState`, `animateColorAsState`, …) |
| Coordinated multi-property | `updateTransition` |
| Material navigation patterns | accompanist/androidx Material motion (`materialSharedAxis*`, container transform helpers) where available; otherwise compose `AnimatedContent` with the matching spec |
| Predictive back | `PredictiveBackHandler` |

For XML/Views, use `MaterialContainerTransform`, `MaterialSharedAxis`, `MaterialFadeThrough` with
the Fragment transition APIs, and `TransitionManager.beginDelayedTransition` for layout changes.

## How motion specs flow through the handover chain

Motion travels end-to-end through the same artifacts as every other spec (D14):

1. **REDESIGN proposal** (`handovers/REDESIGN_PROPOSAL_v<N>.md`, prompt 07) — each redesigned screen
   carries a one-line **Motion** entry: the transition into/out of the screen, key in-screen
   animations, and predictive-back behavior. Drawn from this doc's decision table against the Pass 8
   gap.
2. **IMPLEMENTATION handover** (`handovers/IMPLEMENTATION_HANDOVER_v<N>.md`, prompt 08) — the
   proposal's Motion lines become the **§Motion Specification** table:

   | Context | Transition | Duration | Easing | Reduced-motion fallback |
   |---|---|---|---|---|
   | List → detail (SCR-003→SCR-008) | Container transform | Long (~500 ms) | Emphasized | Instant swap, no transform |
   | Tab A ↔ Tab B | Shared axis X | Medium (~300 ms) | Standard | Instant content swap |

   Per the IMPL schema, every proposal Motion line maps to a ticket or is explicitly rejected (D15
   V-rule).
3. **Ticket** — the implementation ticket for the screen carries a `Motion:` field pointing at its
   row(s) in the Motion Specification.
4. **Checklist** — `07-checklists/design-modernization.md` has a motion item that verifies the built
   motion against the IMPLEMENTATION handover's §Motion Specification (and the reduced-motion
   behavior against `07-checklists/accessibility.md`). Post-build verification is performed by
   `03-design/12-design-qa.md`.

## Acceptance bar (every animation, before declaring done)

- [ ] Has a **duration token** from the M3 scale (short/medium/long; ~50–600 ms) — no orphan magic
      durations.
- [ ] Has an explicit **easing** curve (standard / emphasized / linear, per the rules above).
- [ ] Has a defined **reduced-motion fallback** that degrades gracefully at
      `animator_duration_scale == 0` with no lost state or affordance.
- [ ] Navigation transition matches the **relationship** per the decision table (container transform
      / shared axis / fade through).
- [ ] Predictive-back behavior is stated (custom-animated or explicitly "system default").
- [ ] List motion uses `animateItemPlacement` / item animator; stagger (if any) is first-appearance
      only and capped.
- [ ] The spec appears in the IMPL §Motion Specification table and is referenced by a ticket's
      `Motion:` field.
