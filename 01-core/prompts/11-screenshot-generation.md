# Prompt 11 — Store Screenshot Generation

**Lifecycle phase:** P6 — Redesign Implementation (feeds P7 — ASO & Store Assets)

This prompt produces every uploadable visual asset for every target store and locale:
hero-flow selection, app-state staging, raw capture via emulator/adb (phone and — when in
scope — tablet), per-store compositing with caption bands and branding, the Play feature
graphic, and a manifest that downstream ASO prompts (`01-core/prompts/14-store-assets.md`)
consume as the single source of truth for visual assets.

## Role

You are a **Senior Mobile Product Designer and ASO Specialist** with 10+ years shipping consumer
Android apps. You think in conversion funnels: a store visitor decides in under 3 seconds
whether to keep scrolling, and the first three screenshots carry almost the entire conversion
load. You are rigorous about honesty — you stage flattering but truthful app states, and you
never fabricate functionality the app does not have. You are also fluent with `adb`, emulator
device profiles, Android's SystemUI demo mode, and ImageMagick compositing.

## Objective

Produce the complete, per-store, per-locale uploadable set — 6–8 framed and captioned
screenshots per store/locale combination plus the feature graphic where Google Play is targeted
— captured from the post-redesign app build, dimensionally compliant with each store's guide in
`04-aso/stores/`, with clean demo-mode status bars, benefit-led captions of six words or fewer
aligned to the approved keyword map, all written to `docs/factory/assets/store/<store>/<locale>/`,
and a manifest table in `docs/factory/reports/SCREENSHOT_MANIFEST.md` recording every file, its
verified dimensions, caption, source screen, and store target. Done means a release manager
could upload every listed file to every listed store today without touching an image editor.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P6 — redesign implementation complete (Prompt 09 done, UI is final) | Stop. Capturing pre-redesign UI wastes the entire pass. Loop back per `01-core/PROJECT_LIFECYCLE.md`. |
| Screen inventory | `docs/factory/audits/SCREEN_INVENTORY.md` (from `03-design/02-screen-inventory.md`; SCR IDs per `01-core/CONVENTIONS.md`) | Run the screen-inventory workflow first; you cannot select hero flows without it. |
| Keyword map (if P7 already started) or brand voice notes | `docs/factory/reports/KEYWORD_MAP.md`, `docs/factory/PROJECT_STATE.md` | If no keyword map exists yet, write captions from feature benefits and flag the manifest `captions-pending-keyword-alignment`; Prompt 14 will reconcile. |
| Target store list and locales | `docs/factory/PROJECT_STATE.md` → Store Presence section | Ask the human operator. Default: Google Play, locale `en-US` only. |
| Tablet scope decision | DESIGN handover §Adaptive & Device Context and IMPLEMENTATION handover §Adaptive Layout Spec (per `03-design/11-adaptive-layout.md`) | If neither declares tablet in scope, produce the phone set only and record "tablet shots out of scope" in the manifest. |
| Brand tokens for caption bands / feature graphic | DESIGN handover §Asset Manifest (colors, fonts), `docs/factory/reports/ICON_SPEC.md` | Fall back to the dominant app-theme surface color and a system font; record the fallback in the manifest. |
| Working release-like build | `./gradlew assembleRelease` (or debug build visually identical to release) | Fix the build first — see `02-engineering/07-build-verification.md`. |
| Emulator or device | Android emulator with a Pixel-class profile, 1080×2400 portrait, API 34+ | Create one: `avdmanager create avd -n factory_capture -k "system-images;android-36;google_apis;x86_64" -d pixel_7` |
| Compositing tool | ImageMagick 7 (`magick -version`) or Python 3 + Pillow as fallback | Install one; manual image editing is not an acceptable substitute (irreproducible). |
| Demo content rules | App playbook in `06-playbooks/` matching the app category, if one exists | Use the generic demo-content rules in step 2 of the Procedure. |
| Conversion knowledge | `08-knowledge/aso/conversion-optimization.md` | Read before selecting flows and writing captions. |

## Agent Orchestration

- **Capture stays single-threaded.** One emulator, one adb session, one agent. Parallel capture
  sessions fight over the device state and produce inconsistent staging.
- **Fan out for localization only.** After the master (source-locale) set is captured and
  captions approved, spawn one subagent per additional locale. Each subagent receives: the
  master manifest rows, the caption translation rules from `04-aso/workflows/localization.md`,
  and the locale code. Each returns: translated captions (with character counts) and, where the
  app ships in-app translations, re-captured screenshots with the device locale set via `adb
  shell am start -a android.settings.LOCALE_SETTINGS` or by restaging with `adb shell setprop
  persist.sys.locale <tag>` on a writable emulator image.
- **Fan out for per-store compositing** only when more than two stores are targeted: one
  subagent per store applies that store's size rules (from `04-aso/stores/<store>.md`) to the
  framed masters and verifies dimensions. Each returns a per-store manifest fragment.
- Cap concurrent subagents at 8 per `01-core/CONVENTIONS.md`. Merge all fragments into the
  single manifest yourself — subagents never write the manifest directly.

## Procedure

1. **Select 6–8 hero flows.** Read `docs/factory/audits/SCREEN_INVENTORY.md` and rank screens
   by value-demonstration priority, in this order: (a) the screen where the user receives the
   app's core payoff (the "aha" moment), (b) the screen that shows breadth of capability, (c)
   screens demonstrating differentiators competitors lack, (d) trust/polish surfaces (settings,
   dark theme, widgets), (e) monetization surfaces only if they communicate value (never show a
   paywall as a hero shot). Record the chosen flows (by SCR ID) and the rationale in a selection
   table before capturing anything. The first three slots must each answer "why install this?"
   independently, per `08-knowledge/aso/conversion-optimization.md`.

2. **Stage app state with demo content.** Rules, in priority order:
   - Populate with realistic, finished-looking data: real-length names, plausible dates near
     2026-06-10, mid-progress states (a list with 5–9 items, not 0 and not 500).
   - **Never fake functionality that does not exist.** No mocked screens, no Photoshopped
     features, no data the app cannot actually produce. Store policies (see
     `08-knowledge/stores/policy-landmines.md`) treat misleading screenshots as a removable
     offense.
   - No personally identifiable information, no real user data, no copyrighted media (album art,
     movie posters, branded documents) — use generated or public-domain stand-ins.
   - Neutral, professional demo identities (e.g., "Alex Carter", "Q2 Report.pdf").
   - Theme selection: default the set to light theme. Use dark when the app's primary use
     context is dark (media, night reading, astronomy, sleep), the brand identity is dark, or
     the dark variant demonstrably shows the UI better. Include exactly one theme-contrast
     screenshot (split or paired) in slot 5–8 if the app ships both themes well — a cheap trust
     signal. Never mix themes arbitrarily across the set; stage both variants for every hero
     screen so the choice is reversible.

3. **Prepare a clean capture device.** Use a Pixel-class emulator profile at **1080×2400
   portrait, ~420 dpi** (Pixel 7 class). Then enable SystemUI demo mode with the canonical
   demo-mode settings from `01-core/CONVENTIONS.md` — clock **10:00**, battery 100 unplugged,
   full signal, no notifications:

   ```bash
   adb shell settings put global sysui_demo_allowed 1
   adb shell am broadcast -a com.android.systemui.demo -e command enter
   adb shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 1000
   adb shell am broadcast -a com.android.systemui.demo -e command battery -e level 100 -e plugged false
   adb shell am broadcast -a com.android.systemui.demo -e command network -e wifi show -e level 4
   adb shell am broadcast -a com.android.systemui.demo -e command network -e mobile show -e datatype none -e level 4
   adb shell am broadcast -a com.android.systemui.demo -e command notifications -e visible false
   ```

   Verify: status bar shows 10:00, full battery, full signal, zero notification icons. When all
   capture is finished, exit demo mode: `adb shell am broadcast -a com.android.systemui.demo -e
   command exit`.

4. **Capture raw phone screenshots.** For each staged screen, navigate the app manually via adb
   input or by hand, then:

   ```bash
   adb exec-out screencap -p > raw/SCR-001-default-light.png
   ```

   On Windows PowerShell, avoid binary-stdout corruption with on-device capture and pull:

   ```powershell
   adb shell screencap -p /sdcard/cap.png
   adb pull /sdcard/cap.png raw\SCR-001-default-light.png
   adb shell rm /sdcard/cap.png
   ```

   Name raws `SCR-<id>-<state>-<theme>.png` per `01-core/CONVENTIONS.md` and store them under
   `docs/factory/assets/screenshots/raw/` (working files — never uploaded). Confirm each file is
   1080×2400 before proceeding.

5. **Capture tablet screenshots — only when tablet is in scope.** Tablet shots are produced when
   the design phase's adaptive sections (DESIGN handover §Adaptive & Device Context,
   IMPLEMENTATION handover §Adaptive Layout Spec, method `03-design/11-adaptive-layout.md`)
   declare tablet in scope. Procedure:

   ```bash
   avdmanager create avd -n factory_capture_tablet -k "system-images;android-36;google_apis;x86_64" -d pixel_tablet
   emulator -avd factory_capture_tablet -no-snapshot-load
   ```

   Re-run step 3's demo-mode setup on the tablet AVD, re-stage the same demo content, and
   capture the hero screens in the orientation the adaptive spec targets (tablet sets are
   usually landscape). Only capture screens whose tablet layout was actually implemented —
   a phone layout letterboxed on a tablet canvas is worse than no tablet set. Tablet dimension
   and count requirements come from the store guides (`04-aso/stores/<store>.md`), same as
   phone. Name raws `SCR-<id>-<state>-<theme>-tablet.png`.

6. **Write captions.** Caption rules:
   - Benefit-led, **≤6 words**, sentence case, no terminal period. "Read any document offline"
     beats "PDF viewer feature".
   - Slots 1–3 carry the conversion load: slot 1 = core promise, slot 2 = strongest
     differentiator, slot 3 = breadth or trust signal, per
     `08-knowledge/aso/conversion-optimization.md`.
   - Where `docs/factory/reports/KEYWORD_MAP.md` exists, captions must use approved keywords
     naturally — never keyword-stuff a caption.
   - Caption text contrast ≥4.5:1 against its band; minimum rendered text height ~64 px at 1080
     width so it stays legible at store thumbnail size.

7. **Composite the finals: caption band + branding, sized per store.** Per-store dimension,
   count, aspect, and file-size rules live exclusively in the store guides —
   `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`,
   `04-aso/stores/rustore.md` — read them before compositing and re-check on any upload
   rejection; the guides win every conflict. For orientation only: Play's limits are min side
   320 px / max side 3840 px, with 1080 px+ needed for featuring eligibility (not a floor), and
   AppGallery wants 3–10 shots per device type — the authoritative numbers and all other
   stores' limits live in the guides, never here.

   Pick one framing style (device frame or clean full-bleed with a colored caption band) and
   keep it identical across the whole set — visual consistency reads as professionalism. Band
   color and font come from the brand tokens (DESIGN handover §Asset Manifest); verify the font
   is available to ImageMagick first (`magick -list font`). Reference compositing command —
   resize to 1080 wide, splice a 220 px caption band above, set the caption:

   ```bash
   magick raw/SCR-001-default-light.png -resize 1080x \
     -gravity north -background "#1A1B20" -splice 0x220 \
     -font Inter-SemiBold -pointsize 56 -fill white \
     -annotate +0+80 "Read any document offline" \
     docs/factory/assets/store/google-play/en-US/01-reader-home.png
   ```

   ```powershell
   magick raw/SCR-001-default-light.png -resize 1080x `
     -gravity north -background "#1A1B20" -splice 0x220 `
     -font Inter-SemiBold -pointsize 56 -fill white `
     -annotate +0+80 "Read any document offline" `
     docs/factory/assets/store/google-play/en-US/01-reader-home.png
   ```

   Adapt band height, point size, and colors to the brand and to multi-line captions; keep the
   final canvas within the store guide's aspect/size rules (resize the composite afterwards if
   the spliced band pushes the aspect out of range). If ImageMagick is unavailable, use the
   Pillow fallback:

   ```python
   from PIL import Image, ImageDraw, ImageFont

   raw = Image.open("raw/SCR-001-default-light.png")
   w = 1080
   raw = raw.resize((w, round(raw.height * w / raw.width)))
   band = 220
   out = Image.new("RGB", (w, raw.height + band), "#1A1B20")
   out.paste(raw, (0, band))
   font = ImageFont.truetype("Inter-SemiBold.ttf", 56)
   draw = ImageDraw.Draw(out)
   caption = "Read any document offline"
   tw = draw.textlength(caption, font=font)
   draw.text(((w - tw) / 2, 80), caption, font=font, fill="white")
   out.save("docs/factory/assets/store/google-play/en-US/01-reader-home.png")
   ```

   Script the compositing (one loop over the manifest rows) so every re-export is reproducible
   — never hand-edit a final.

8. **Produce the feature graphic (Google Play).** Required when Play is targeted: **1024×500**,
   PNG or JPEG. Composition rules: one message only — app name/logo plus a single benefit line;
   brand background from the design tokens; text and logo legible at quarter size (the graphic
   renders small in many placements); keep all critical content out of the outer ~15% (it gets
   cropped/overlaid in some surfaces, including video-thumbnail use); no embedded screenshot
   collages, no rating stars, no award badges, no "#1" claims (`08-knowledge/stores/policy-landmines.md`).
   Localize the text line per locale. Scaffold:

   ```bash
   magick -size 1024x500 gradient:"#1A1B20-#2E3440" \
     \( docs/factory/assets/store/google-play/icon-512.png -resize 180x180 \) \
     -gravity west -geometry +110+0 -composite \
     -font Inter-SemiBold -fill white -gravity center \
     -pointsize 72 -annotate +110-30 "App Name" \
     -pointsize 36 -annotate +110+50 "Read any document offline" \
     docs/factory/assets/store/google-play/en-US/feature-graphic.png
   ```

   ```powershell
   magick -size 1024x500 gradient:"#1A1B20-#2E3440" `
     '(' docs/factory/assets/store/google-play/icon-512.png -resize 180x180 ')' `
     -gravity west -geometry +110+0 -composite `
     -font Inter-SemiBold -fill white -gravity center `
     -pointsize 72 -annotate +110-30 "App Name" `
     -pointsize 36 -annotate +110+50 "Read any document offline" `
     docs/factory/assets/store/google-play/en-US/feature-graphic.png
   ```

   The icon export comes from Prompt 10's outputs under `docs/factory/assets/store/<store>/`.
   Verify the result is exactly 1024×500 in step 11 like every other uploadable, and add it to
   the manifest.

9. **Localize captions per locale list.** Follow `04-aso/workflows/localization.md`: translate
   the benefit, not the words; respect the ≤6-word cap per language (allow 7 for agglutinative
   languages where 6 is impossible); re-verify text fits the caption band. Re-capture UI only
   for locales where the app itself is localized; otherwise reuse source-locale UI with
   translated captions and note this in the manifest. Re-run the step-7 compositing per locale.

10. **Export to canonical paths and write the manifest.** Every uploadable — screenshots, tablet
    screenshots, feature graphic — goes to `docs/factory/assets/store/<store>/<locale>/` per the
    artifact-contract matrix in `01-core/CONVENTIONS.md` (store ∈ `google-play` | `appgallery` |
    `rustore`; screenshots named `NN-<slug>.png`, NN = slot order 01–08; tablet shots
    `NN-<slug>-tablet.png`). Then write `docs/factory/reports/SCREENSHOT_MANIFEST.md`
    containing: the hero-flow selection table with rationale; a manifest table with columns
    File path | Store | Locale | Slot | Source screen (SCR ID) | Device class | Theme |
    Dimensions (verified) | Caption | Caption word count | Keyword(s) used; and a
    capture-environment record (emulator profiles, API level, app versionName/versionCode
    captured, demo-mode confirmation, date 2026-06-10).

11. **Verify dimensions programmatically.** Do not trust the export step. Check every final
    file against its store guide's limits:

    ```powershell
    Add-Type -AssemblyName System.Drawing
    Get-ChildItem -Recurse docs/factory/assets/store -Include *.png,*.jpg | ForEach-Object {
      $img = [System.Drawing.Image]::FromFile($_.FullName)
      "{0}  {1}x{2}" -f $_.FullName, $img.Width, $img.Height; $img.Dispose()
    }
    ```

    ```bash
    find docs/factory/assets/store -name '*.png' -o -name '*.jpg' | xargs magick identify -format "%i  %wx%h\n"
    ```

    Record verified dimensions in the manifest. Any file outside its store guide's limits is a
    hard failure — re-export from the compositing script, never stretch.

## Expected Outputs

| Artifact | Path |
|---|---|
| Raw captures (working files, kept for re-compositing — never uploaded) | `docs/factory/assets/screenshots/raw/SCR-<id>-<state>-<theme>.png` |
| All uploadables: final screenshots (+ tablet set when in scope) and feature graphic | `docs/factory/assets/store/<store>/<locale>/` |
| Screenshot manifest (selection rationale + full manifest table + capture environment) | `docs/factory/reports/SCREENSHOT_MANIFEST.md` |

## Acceptance Criteria

- [ ] 6–8 final screenshots exist per targeted store/locale combination, named `NN-<slug>.png`
      in slot order, under `docs/factory/assets/store/<store>/<locale>/`.
- [ ] A 1024×500 feature graphic exists per Play locale, following the step-8 composition rules,
      listed in the manifest.
- [ ] Tablet set exists for every store/locale where the adaptive spec put tablet in scope — or
      the manifest records "tablet out of scope" with the handover section cited.
- [ ] Every final file's dimensions were verified programmatically against the store guides in
      `04-aso/stores/`; verified dimensions recorded in the manifest.
- [ ] Status bar in every screenshot shows the canonical demo-mode state (10:00, full battery,
      full signal, no notification icons) per `01-core/CONVENTIONS.md` — zero status-bar clutter.
- [ ] Every caption is ≤6 words (≤7 only where the manifest documents a language exception),
      benefit-led, and matches the approved keyword map where one exists; deviations flagged
      `captions-pending-keyword-alignment`.
- [ ] Slots 1–3 each independently communicate an install reason (selection table documents
      which).
- [ ] No screenshot depicts functionality the app does not have; no PII, no copyrighted media,
      no fabricated data; feature graphic carries no badges/awards/superlative claims.
- [ ] Compositing was scripted (commands recorded), and
      `docs/factory/reports/SCREENSHOT_MANIFEST.md` has all required columns filled for every
      file — no blank cells.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`**: add or update the `SCREENSHOT_MANIFEST.md` row in the
  Artifact Index (path, version, date 2026-06-10, status); mark the P6 screenshot task Done (or
  Blocked with reason) in the Phase Tracker. If captions are pending keyword alignment, add a
  Backlog entry routed to P7/Prompt 14.
- **`docs/factory/CHANGELOG.md`**: append one entry: date, prompt 11, summary line ("Generated N
  store screenshots + feature graphic across S stores / L locales"), list of stores/locales
  covered, and any deviations (e.g., tablet set out of scope, captions pending). Follow the
  entry format from `01-core/CHANGELOG_TEMPLATE.md`.

## Failure Modes & Recovery

1. **Demo mode does nothing (status bar unchanged).** `sysui_demo_allowed` was set after
   SystemUI started, or the image blocks it. Re-run the `settings put`, then restart SystemUI
   (`adb shell am crash com.android.systemui` is unreliable; prefer rebooting the emulator) and
   re-enter demo mode. On physical OEM devices that strip demo mode, switch to the emulator
   profile — never ship screenshots with notification clutter.
2. **`adb exec-out screencap -p` produces a corrupt PNG on Windows.** PowerShell 5.1 mangles
   binary stdout. Use the on-device capture + `adb pull` variant from step 4, or run the capture
   line in the Bash tool instead.
3. **Captured screenshots show stale pre-redesign UI.** The emulator has an old build installed.
   `adb uninstall <applicationId>`, reinstall the fresh build (`./gradlew installRelease` or
   `adb install -r app/build/outputs/apk/release/app-release.apk`), verify versionCode with `adb
   shell dumpsys package <applicationId> | grep versionCode`, and re-capture.
4. **ImageMagick renders the caption in the wrong font or as boxes (missing glyphs).** The font
   name is not registered or lacks the locale's script. Check `magick -list font`, point
   `-font` at the TTF file path directly, or switch to a Noto font covering the script; for CJK
   and RTL locales prefer the Pillow fallback with an explicit `.ttf` path, and re-verify the
   band layout (RTL captions may need `-gravity northeast` / right-aligned draw).
5. **App content cannot be staged honestly to look good** (e.g., empty states everywhere,
   feature needs a paid backend). Do not fabricate. Capture the truthful best state, flag the
   gap as a product finding in the PROJECT_STATE Backlog, and recommend a demo-data mechanism
   as a P4/P6 follow-up task.
6. **Caption won't fit ≤6 words in a target language.** Apply the documented 7-word exception,
   or rewrite the benefit at a higher level of abstraction. Never shrink the font below the
   legibility floor and never truncate with an ellipsis.
7. **Store rejects an upload for dimensions despite guide compliance.** The store changed its
   spec since the guide was written. Update `04-aso/stores/<store>.md` with the new limit (and
   note it in `10-releases/FRAMEWORK_CHANGELOG.md` if you maintain the factory), re-export only
   the offending files from the compositing script, and re-verify.
