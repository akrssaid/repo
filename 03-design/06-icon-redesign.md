# Icon Redesign Method

This file is the **SOLE procedure** for auditing, redesigning, producing, and validating a target
app's launcher and store icon (canonical decision D12). `01-core/prompts/10-icon-redesign.md`
keeps only role, orchestration, and outputs and **defers entirely to this method** — no icon
dimensions, brief fields, or validation steps are restated there. Brand and visual principles live
in `08-knowledge/design/app-icon-principles.md`; this file is the operational procedure: what to
inventory, how to analyze the competitive shelf, what the adaptive icon spec demands dimensionally,
how to ship every required asset, and how to validate the result.

Conventions (IDs, scales, severities, canonical paths) are defined once in
`01-core/CONVENTIONS.md` and cited here, not redefined. The brief and every design decision are
recorded in `docs/factory/reports/ICON_SPEC.md` (the artifact named for prompt 10 in the D5
artifact-contract matrix) — **not embedded in the DESIGN handover**; the handover's Asset Manifest
references `ICON_SPEC.md` and the produced assets. The store-side conversion loop (A/B testing) is
owned by `04-aso/workflows/icon-optimization.md`; this file produces the icon, that file tests it.

---

## 1. Current Icon Audit

Run this inventory before any design work. Record results in
`docs/factory/audits/DESIGN_AUDIT.md` under an "Icon Inventory" heading.

### 1.1 Asset inventory procedure

1. Locate all launcher icon resources:

   ```powershell
   Get-ChildItem -Recurse app\src\main\res -Include "ic_launcher*","*launcher*" |
     Select-Object FullName, Length
   ```

2. Fill the inventory table:

   | Item | Expected location | Present? | Notes |
   |---|---|---|---|
   | mdpi raster | `res/mipmap-mdpi/ic_launcher.webp` (48×48 px) | | |
   | hdpi raster | `res/mipmap-hdpi/ic_launcher.webp` (72×72 px) | | |
   | xhdpi raster | `res/mipmap-xhdpi/ic_launcher.webp` (96×96 px) | | |
   | xxhdpi raster | `res/mipmap-xxhdpi/ic_launcher.webp` (144×144 px) | | |
   | xxxhdpi raster | `res/mipmap-xxxhdpi/ic_launcher.webp` (192×192 px) | | |
   | Round variants | `res/mipmap-*/ic_launcher_round.webp` | | Legacy API 25 round mask |
   | Adaptive XML | `res/mipmap-anydpi-v26/ic_launcher.xml` | | See 1.2 |
   | Foreground layer | drawable or mipmap referenced by adaptive XML | | Vector or raster? |
   | Background layer | drawable, mipmap, or `@color/...` | | Solid color preferred |
   | Monochrome layer | `<monochrome>` element in adaptive XML | | Android 13+ themed icons |
   | Store icon source | 512×512 px PNG, 32-bit, no alpha-trimmed edges | | Often only in Play Console — recover or recreate |
   | Vector master | SVG/Figma/AI source file | | If absent, flag: redesign must produce one |

3. Check the manifest wiring: `android:icon` and `android:roundIcon` in `AndroidManifest.xml`
   must point at `@mipmap/ic_launcher` / `@mipmap/ic_launcher_round`.

### 1.2 Adaptive icon dissection

Open `res/mipmap-anydpi-v26/ic_launcher.xml` and record:

- **Background**: solid color (`@color/...`), simple gradient drawable, or complex art? Complex
  backgrounds are an audit finding (parallax artifacts, mask clipping).
- **Foreground**: vector drawable (good) or raster (acceptable, flag if low-res)? Does content
  stay inside the 66dp safe zone (Section 4)?
- **Monochrome**: present? If absent on a 2026 release, log a DES finding (Medium): themed icons
  on Android 13+ fall back to a tinted full icon or a generic treatment, which looks broken next
  to compliant apps.

### 1.3 Common audit findings (log with `DES-NNN` IDs per `03-design/01-design-audit.md`; format in `01-core/CONVENTIONS.md`)

| Finding | Typical severity |
|---|---|
| No adaptive icon (raster-only, pre-API-26 style) | High |
| Foreground content clipped by circle mask | High |
| Missing monochrome layer | Medium |
| Store icon ≠ launcher icon (brand mismatch) | Medium |
| No vector master recoverable | Medium |
| Legacy `mipmap-*dpi` rasters stale vs adaptive XML | Low |

---

## 2. Competitive Shelf Analysis

The icon competes on a shelf, not in isolation. Before sketching, map the shelf.

1. Open Google Play, search the app's primary keyword (from
   `04-aso/workflows/keyword-research.md` output if available; otherwise the category top chart).
2. Capture the **top 10 icons** in search results — screenshot the results page so icons appear at
   real shelf size, and save it as evidence alongside the brief in
   `docs/factory/reports/` (reference it from `ICON_SPEC.md`; the abolished `assets/icons/` path
   is not used).
3. Build the cluster table:

   | # | App | Dominant color | Shape language | Metaphor | Style |
   |---|---|---|---|---|---|
   | 1 | … | e.g. blue | rounded glyph on solid bg | document | flat |

4. Identify clusters: if 7/10 icons are blue with a white glyph, blue-with-white-glyph is the
   **conformity zone**. Conformity signals category membership; differentiation wins the eye.
5. Define the **differentiation gap**: at least one axis (color, shape, metaphor) where the new
   icon can stand apart while still reading as category-native. Record one sentence:
   "Differentiate on X, conform on Y."

---

## 3. Design Brief

Produce a one-page brief in `docs/factory/reports/ICON_SPEC.md` before production — this file is
the single home for the icon brief and every design decision (D12). The DESIGN handover does
**not** embed the brief; its Asset Manifest links to `ICON_SPEC.md`. Required fields:

| Field | Content |
|---|---|
| Brand constraints | Mandatory colors/marks from the design system extracted in `03-design/04-design-system-extraction.md`; what may NOT change |
| Shelf evidence | The shelf capture and cluster table from Section 2 (or its reference) |
| Metaphor candidates | 2–4 candidate symbols, each one noun ("shield", "play triangle", "folder") |
| Shelf positioning | The differentiation/conformity sentence from Section 2 |
| Silhouette test | Each candidate must be recognizable as a pure black silhouette at 48×48 px. Fill the shape black, scale down, judge. Reject candidates that fail. |
| Scalability test | Candidate must survive 48 px (launcher shelf), 36 px (notification context), and 16 px-equivalent (settings lists) without detail collapse. Max ~2 visual elements. |
| Selected direction | One candidate, one sentence of rationale |

Rules of thumb (full rationale in `08-knowledge/design/app-icon-principles.md`):

- One metaphor. Icons with two ideas read as zero ideas at 48 px.
- No text in the icon. Single letters are acceptable only as the established brand mark.
- No photographic or high-detail art; flat or lightly dimensional shapes only.

---

## 4. Adaptive Icon Specification (exact dimensions)

Both layers are **108×108 dp**. The launcher masks and may move them.

| Zone | Size | Rule |
|---|---|---|
| Full layer canvas | 108×108 dp | Both foreground and background fill this entirely |
| Visible area (typical mask) | ~72×72 dp center | Outer 18 dp per side is reserved for parallax/mask |
| Safe zone | 66 dp diameter circle, centered | ALL critical foreground content must fit here |

- **Mask survival**: content must survive circle, squircle, rounded-square, and teardrop masks.
  Anything outside the 66 dp circle WILL be clipped on some launchers. Design to the circle;
  everything else is a superset.
- **Background layer**: solid color or very simple gradient, edge-to-edge fill, no critical
  detail. Launchers parallax-shift layers up to ~9 dp; a busy background shimmers and tears.
- **Foreground layer**: the metaphor glyph, centered in the safe zone, with transparency around
  it. Never bake a background shape (circle/rounded square plate) into the foreground — the mask
  provides the shape.
- **Monochrome layer**: single-color vector (`android:fillColor` value is ignored — the system
  tints it), same 66 dp safe-zone rule, typically the foreground glyph simplified to one flat
  shape. **Minimum stroke weight is ~2 dp** — this is the authoritative floor; strokes thinner
  than ~2 dp disappear when the system tints and scales the layer. Prefer filled shapes over
  strokes where possible.

---

## 5. Production Pipeline

1. **Vector master.** Produce/obtain an SVG master of the glyph on a 108×108 dp artboard with the
   66 dp safe circle as a guide layer. Store the master alongside the brief and reference it from
   `docs/factory/reports/ICON_SPEC.md` (the abolished `assets/icons/` path is not used).
2. **Layer drawables.** Convert to Android vector drawables (Android Studio: File → New → Vector
   Asset, or `vd-tool`): `drawable/ic_launcher_foreground.xml`,
   `drawable/ic_launcher_monochrome.xml`; background as `values/colors.xml` entry
   `ic_launcher_background` (preferred) or a simple drawable.
3. **Adaptive XML.** Write `res/mipmap-anydpi-v26/ic_launcher.xml` (and an identical
   `ic_launcher_round.xml`):

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
       <background android:drawable="@color/ic_launcher_background"/>
       <foreground android:drawable="@drawable/ic_launcher_foreground"/>
       <monochrome android:drawable="@drawable/ic_launcher_monochrome"/>
   </adaptive-icon>
   ```

4. **Legacy rasters.** Generate all `mipmap-*dpi` WebP/PNG exports for pre-26 devices and
   fallback contexts. Use Android Studio's **Image Asset wizard** (right-click `res` → New →
   Image Asset → Launcher Icons (Adaptive and Legacy)) — it emits every density plus round
   variants from the layer sources in one pass. Sizes: 48/72/96/144/192 px (m/h/x/xx/xxxhdpi).
5. **Store icon.** Export a **512×512 px, 32-bit PNG** from the vector master: full-bleed
   background (the masked launcher look, but you control the corner radius — Play applies its own
   ~20% rounding, so design full-square and let Play round it). Max 1 MB. Store-icon exports are
   uploadable assets and go to `docs/factory/assets/store/<store>/` (e.g.
   `assets/store/google-play/icon-512.png`), one per target store per D5 — never the abolished
   `assets/icons/` path. AppGallery/RuStore store icons that differ in spec live under their own
   `<store>` folder.
6. Commit; update `docs/factory/CHANGELOG.md`.

---

## 6. Validation Protocol

All steps must pass before the icon work is Done.

1. **Build & install**: `./gradlew :app:installDebug`, confirm new icon on the launcher.
2. **Mask check (emulator)**: on a Pixel emulator, long-press home → Styles & wallpapers (or
   developer launcher settings) and cycle icon shapes: circle, squircle, rounded square, teardrop.
   No clipped content, no off-center glyph in any shape.
3. **Themed icon toggle (Android 13+ emulator)**: wallpaper & style → "Themed icons" ON. The
   monochrome layer must render as a clean tinted glyph, not a gray blob or fallback tint.
4. **Shelf legibility**: place a screenshot of the icon at 48×48 px next to the Section 2 shelf
   capture. It must be identifiable in under one second and distinct from every cluster neighbor.
5. **Parallax sanity**: tilt/scroll the launcher (or drag the icon); no visible tearing between
   layers.
6. **Store A/B handoff**: do not assume the new icon converts better, and do not design the test
   here. The A/B test plan is owned by `04-aso/workflows/icon-optimization.md` plus the
   experiment-program workflow (`04-aso/workflows/experiment-program.md`), which assigns the
   `EXP-NNN` ID and logs results to `docs/factory/reports/EXPERIMENT_LOG.md`. This method's only
   obligation is to hand off a production-ready icon and note the constraint that the launcher icon
   and store icon must remain visually identical through any test, so console experiments measure
   the icon and not a mismatch.

### Acceptance checklist

- [ ] Inventory table complete; all findings logged with `DES-NNN` IDs (per `01-core/CONVENTIONS.md`)
- [ ] Shelf analysis captured with cluster table and differentiation sentence in `ICON_SPEC.md`
- [ ] Brief recorded in `docs/factory/reports/ICON_SPEC.md` (not the handover) and approved (silhouette + scalability tests passed)
- [ ] Monochrome layer authored with ≥ 2 dp minimum stroke weight
- [ ] Adaptive XML has background, foreground, AND monochrome layers
- [ ] All five density rasters + round variants regenerated
- [ ] 512×512 store icon exported to `assets/store/<store>/` per target store, matching the launcher icon
- [ ] All six validation steps pass on emulator
- [ ] DESIGN handover Asset Manifest references `ICON_SPEC.md` and the produced assets
- [ ] Store A/B test handed to `04-aso/workflows/icon-optimization.md` (EXP ID assigned), not designed here
