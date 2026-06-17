# App Icon Principles

Doctrine for designing and judging app icons across the portfolio. The icon is the single
highest-leverage pixel real estate the app owns: it appears in every store impression, every
launcher glance, every notification. This doc holds the *principles*; the production procedure is
`03-design/06-icon-redesign.md` and the testing/shipping workflow is
`04-aso/workflows/icon-optimization.md`. Composition doctrine here is stable; adaptive-icon
technical specs and Play asset requirements drift slowly — **verify against current official
docs** when executing.

**last reviewed: 2026-06-10**

---

## The Icon's Three Jobs

1. **Shelf differentiation.** In a Play search results row or "similar apps" strip, the icon is
   the only asset every competitor also shows at the same size. Job one: be findable in a row of
   eight near-identical metaphors. Differentiation drives tap-through before a single screenshot
   is seen.
2. **Instant recognition.** On the user's launcher among 60+ icons, the user must find the app in
   under a second by silhouette and color alone — recognition is pre-attentive (shape + color),
   not read. Redesigns that break recognition for existing users have a real retention cost;
   evolve, don't replace, for apps with large installed bases.
3. **Quality signal.** Users infer the app's internal quality from the icon before installing.
   A gradient-soup, clip-art, or 2014-skeuomorphic icon says "abandoned"; a crisp current-style
   icon says "maintained". For utility apps handling user files, this is a trust signal
   (see `08-knowledge/design/mobile-ux-principles.md` §9).

Every icon decision is judged against these three jobs — in that order for acquisition-focused
apps, recognition-first for mature apps with loyal bases.

---

## Composition Rules

| Rule | Rationale |
|---|---|
| **One metaphor** | One object, one idea. A folder *and* a magnifier *and* a gear is three icons fighting in one square; nothing reads at small sizes |
| **Strong silhouette** | The shape must be identifiable from its outline alone (squint test / threshold to black). Centered, simple geometry beats detailed illustration |
| **2–3 colors maximum** | Plus neutrals. More colors = visual noise at 48px and weaker recognition; one dominant hue should "own" the icon |
| **No text** | Unreadable at launcher sizes, breaks in localization, and Play policy restricts misleading text; single-letter glyphs are allowed but weak unless the brand *is* the letter |
| **No photos** | Photographic content turns to mud below 96px and never matches the launcher's graphic context |
| **No UI screenshots** | Miniature app screens are illegible and read as amateur; the icon is a symbol, not a preview (screenshots do previews — `04-aso/workflows/screenshot-optimization.md`) |
| Flat-to-subtle depth | Current Android style: flat or gently shaded forms; heavy bevels, long shadows, and glossy orbs date the app instantly |
| Respect the grid | Use the Material icon-grid proportions (keyline shapes) so optical weight matches neighboring system/app icons |

---

## The Scalability Test

An icon must **read at 48px and shine at 512px**. Test at both ends before any review:

1. Render at 48×48 px on a mid-gray background. The metaphor must be identifiable and the
   silhouette unambiguous. If any element disappears or aliases into noise, simplify — do not
   "sharpen".
2. Render at 512×512 (the Play Store listing asset). At this size the icon must reward attention:
   clean curves, deliberate color transitions, no empty flatness. Subtle gradient or texture that
   vanishes at 48px is *allowed* here — degradation must be graceful, not load-bearing.
3. Check the in-between: 108px (launcher on common densities) and the circular mask crop.

Rule of thumb: design at 512, art-direct at 48. If forced to choose, 48px legibility wins — far
more impressions happen small.

---

## Adaptive Icon — Technical Recap

Full production spec and asset pipeline: `03-design/06-icon-redesign.md`. The facts:

- **Canvas 108×108 dp, safe zone 66 dp** diameter circle. Everything identity-critical lives
  inside the 66dp circle; the outer ring is reserved for mask cropping and the launcher's
  parallax/visual effects.
- **Mask survival**: launchers crop the 108dp canvas to circle, squircle, rounded square, or
  teardrop. The foreground layer must read under *every* mask — verify in Android Studio's
  Asset Studio preview or on devices with different launchers. Background layer must be
  full-bleed (no transparency at edges).
- **Layers**: background (full-bleed color/gradient/pattern, no critical content) + foreground
  (the metaphor, inside safe zone). Do not bake a shadow between layers that fights launcher
  theming.
- **Monochrome/themed layer** (API 33+ themed icons): a single-color, alpha-only version of the
  foreground glyph. It must survive as pure silhouette — this is the squint test made literal.
  Apps without a monochrome layer look broken on themed-icon launchers; treat it as mandatory in
  every redesign.
- The Play Store 512px listing icon is a separate asset and is **not** masked by Play (it applies
  its own corner rounding) — keep it visually consistent with the adaptive icon, not identical
  in geometry.

---

## Category-Convention Awareness

Every utility category has visual clichés. Doctrine: **conform on metaphor, differentiate on
color and style** — users scan for the expected object (job 2 of search: "which of these is a
file manager?") but choose between lookalikes by distinctiveness (job 1).

| Category | Cliché metaphor | Cliché palette | Differentiation room |
|---|---|---|---|
| File manager | Folder | Blue/yellow folder | Keep a folder-derived shape; own an uncommon hue (teal, coral), distinctive corner language or negative-space mark |
| PDF / document reader | Page with folded corner, "PDF" letters | Red (Adobe gravity) | Keep the page; drop the letters; escape red entirely — red is ceded territory |
| Video downloader | Down arrow, play triangle | Red/white (YouTube gravity) | Combine arrow+play into one fused mark; avoid pure YouTube red/white (also a policy-adjacent confusion risk) |
| Data recovery | Clock-rewind arrow, life ring, magnet | Green/blue | Rewind-arrow metaphor is strong — own it with a distinctive gradient and silhouette weight |
| Cleaner / booster | Broom, rocket, shield | Blue/green | Extremely saturated cliché space; style differentiation matters more than metaphor novelty |

When to break metaphor convention: only when the app's brand is already strong enough to carry an
abstract mark, or the category leader owns the metaphor so completely that conforming reads as a
clone. Both are rare in this portfolio — default is conform/differentiate as above.

---

## Store-Context Testing

An icon is never approved in isolation. Required review contexts:

1. **Play surfaces**: against Play's light (white) and dark backgrounds — the icon needs either
   internal contrast or a defined edge on both; a white-background icon dissolves into light mode.
2. **The competitor row**: composite the icon into a real search-results screenshot for the app's
   top keyword between the actual top-5 competitor icons (capture during
   `04-aso/workflows/competitor-analysis.md`). It must be the row's most distinct *without*
   looking off-category.
3. **Launcher reality**: on-device, light/dark wallpaper, themed-icon mode on/off, inside a folder
   grid at smallest size.
4. Notification small-icon derivative legibility (status bar, single color) — designed alongside,
   not as an afterthought.

---

## Versioning Discipline

- **Icon changes are A/B tested, never shipped on instinct.** Run Play Store listing experiments
  per `04-aso/workflows/icon-optimization.md`; a "better looking" icon that drops conversion 5%
  is a worse icon. Ship only on statistically meaningful wins or strategic rebrands with
  accepted, quantified risk.
- One variable at a time: metaphor change, color change, and style change are separate
  experiments — a combined change that wins teaches nothing for the next app.
- Keep every shipped icon version (source + exports) in the target repo under
  `docs/factory/assets/icons/` with dates and experiment results; recognition (job 2) means
  rollback must always be possible.
- Never change the icon in the same release as a major UI redesign if attribution of
  rating/conversion shifts matters — stagger by one release when feasible.

---

## Principles Applied — Three Worked Examples

**File manager.** Candidate metaphors: folder (universal, instant, saturated), drawer/cabinet
(reads "storage" but ambiguous at 48px), abstract grid (clean but says nothing). Choose the
folder — conform on metaphor — then differentiate: a single bold teal folder with a distinctive
notch silhouette, one white negative-space cut suggesting organization, flat with one subtle
shade. Three colors total. Squint test: still a folder. Competitor row of blue/yellow folders:
the teal one wins the glance. Monochrome layer: the notched-folder outline survives as pure shape.

**PDF reader.** Candidate metaphors: page with folded corner (universal), "PDF" lettering
(cliché, fails the no-text rule, unreadable at 48px), book (says reader, loses "document").
Choose the folded-corner page; delete the letters — the fold *is* the document signifier. Escape
red: deep indigo page on a soft warm background, the folded corner as the single highlight
element. The fold must sit inside the 66dp safe zone or circular masks amputate the metaphor's
key feature. At 512px, a faint paper gradient rewards the close look; at 48px it disappears
harmlessly — graceful degradation as designed.

**Data recovery app.** Candidate metaphors: rewind-clock arrow (time reversal — strong, accurate),
magnet (vague, dated), life ring (rescue, but reads "help/support app"). Choose the
counter-clockwise arrow forming a near-circle around a small file/photo glyph — one fused mark,
not two competing objects. Palette: confident green-to-teal (recovery = success/safety) on a dark
background for shelf contrast against the category's light icons. Silhouette test: the broken
circle with arrowhead is unmistakable. Trust check (job 3): precise geometry and calm palette —
this app will touch users' lost photos; the icon must look like a scalpel, not a late-night
infomercial.

---

Cross-references: production pipeline `03-design/06-icon-redesign.md`; experiment workflow
`04-aso/workflows/icon-optimization.md`; conversion context
`08-knowledge/aso/conversion-optimization.md`; playbook-specific guidance in `06-playbooks/`.
