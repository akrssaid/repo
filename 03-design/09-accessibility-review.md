# Accessibility Review Method

This document defines the accessibility review method for a redesigned target app: an automated
pass, a manual protocol with exact thresholds and commands, severity mapping, and a finding
format. Its output feeds the release gate in `07-checklists/accessibility.md` — that checklist is
pass/fail; this file is how you generate the evidence. It is executed during P6 — Redesign
Implementation (by `01-core/prompts/09-ui-implementation.md` after each screen batch) and again in
full during P8 via `01-core/prompts/15-final-verification.md`. Target standard: WCAG 2.1 AA
adapted to Android, per Google's accessibility guidelines.

---

## 1. Automated Pass

Run all three layers; automation catches roughly half of real issues — never skip Section 2.

1. **Accessibility Scanner** (on-device, by Google):
   - Install from Play on the test device/emulator (`com.google.android.apps.accessibility.auditor`).
   - Enable: Settings → Accessibility → Accessibility Scanner → toggle on.
   - For every screen in the screen inventory (`03-design/02-screen-inventory.md` output), open
     the screen, tap the Scanner FAB, choose "Snapshot" (or "Record" for flows with dialogs).
   - Export results (share → save) into `docs/factory/reports/a11y/scanner/`, one capture per
     screen, named after the screen inventory ID.
   - Scanner reports: touch target size, low contrast, missing labels, duplicated labels.
2. **Compose UI test semantics checks** (Compose apps): add an accessibility-assertion pass to
   existing screen tests:

   ```kotlin
   // build.gradle.kts (androidTest): androidx.compose.ui:ui-test-junit4 + ui-test-manifest
   @Test
   fun homeScreen_iconButtons_haveLabels() {
       composeTestRule.setContent { AppTheme { HomeScreen() } }
       composeTestRule.onAllNodes(hasClickAction() and !hasText(""))
           .assertAll(hasContentDescription() or hasText())
   }
   ```

   Additionally enable Espresso's `AccessibilityChecks.enable().setRunChecksFromRootView(true)`
   in View-based instrumentation tests — it runs the same engine as Scanner on every interaction.
3. **Lint a11y rules**: run and triage the accessibility category:

   ```powershell
   ./gradlew :app:lint
   # then inspect app/build/reports/lint-results-*.html, category "Accessibility"
   ```

   Key rule IDs: `ContentDescription` (missing on ImageView/ImageButton), `LabelFor` (EditText
   without label), `ClickableViewAccessibility` (onTouch without performClick),
   `KeyboardInaccessibleWidget`, `SmallSp`. Do not suppress any of these without a written
   justification in the finding log.

---

## 2. Manual Protocol

### 2.1 Contrast

Thresholds (WCAG 2.1 AA):

| Element | Minimum ratio |
|---|---|
| Normal text (< 18 pt / < 14 pt bold) | 4.5 : 1 |
| Large text (≥ 18 pt, or ≥ 14 pt bold) | 3 : 1 |
| UI components & graphical objects (icons, input borders, focus indicators) | 3 : 1 |

Sampling procedure:

1. Use the screenshot set from `03-design/03-screenshot-capture.md` (light AND dark theme sets).
2. For each screen, sample foreground and background pixel pairs with a color picker (Windows:
   PowerToys Color Picker, Win+Shift+C) at: body text on background, secondary/hint text, text on
   primary buttons, icon-only actions, disabled-looking-but-enabled controls.
3. Compute the ratio with any WCAG contrast calculator (e.g. WebAIM) from the two hex values;
   record `#fg / #bg = ratio` in the finding. Sample text mid-stroke, not at anti-aliased edges.
4. Theme-token shortcut: if the app uses the token sheet from
   `03-design/04-design-system-extraction.md`, check each `on*` / surface pair once in the table
   instead of per-screen — then per-screen sampling only covers hardcoded deviations.

### 2.2 Touch targets

Minimum **48×48 dp** for every interactive element (visual glyph may be smaller; the touchable
area may not), with ≥ 8 dp spacing between adjacent targets.

Detection: Android Studio → Layout Inspector (running app) → select each clickable node → check
width/height in dp; for Compose, the semantics tree shows the touch bounds. Quick on-device
check: enable Developer options → Pointer location, tap the control edge-to-edge and read the dp.
Compose note: M3 components enforce 48 dp via `minimumInteractiveComponentSize`; custom
`Modifier.clickable` on small Boxes does not — those are the usual offenders.

### 2.3 Content descriptions

Rules:

| Element type | Rule |
|---|---|
| Informative image / icon-only button | MUST have a description naming the action or content ("Delete download", not "trash icon") |
| Decorative image (dividers, background art, icon duplicating adjacent text) | MUST be explicitly null/`importantForAccessibility="no"` so TalkBack skips it |
| Text already on screen | Never duplicated into a sibling icon's description |
| Stateful controls | Description or stateDescription reflects state ("Favorited" vs "Add to favorites") |

Grep patterns to find candidates (each hit needs a decorative-vs-informative decision, not a
blind fix):

```powershell
# Compose: literal null descriptions (verify each is genuinely decorative)
Get-ChildItem -Recurse app\src -Filter *.kt | Select-String "contentDescription = null"
# Compose: Icon/Image calls — cross-check that informative ones pass a real string
Get-ChildItem -Recurse app\src -Filter *.kt | Select-String "Icon\(|Image\("
# XML: ImageView/ImageButton lacking the attribute
Get-ChildItem -Recurse app\src\main\res\layout -Filter *.xml |
  Select-String "<ImageView|<ImageButton" -Context 0,4 |
  Where-Object { $_.Context.PostContext -notmatch "contentDescription" }
```

### 2.4 TalkBack walkthrough

Enable TalkBack via adb (no menu digging):

```powershell
adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
adb shell settings put secure accessibility_enabled 1
# Disable afterwards:
adb shell settings put secure enabled_accessibility_services ""
adb shell settings put secure accessibility_enabled 0
```

The service component name varies by device, OEM, and Accessibility Suite version — verify the
exact string first and substitute it:

```powershell
adb shell pm list packages | Select-String "talkback|accessibility"
adb shell dumpsys accessibility | Select-String "Service\["
```

Walkthrough script — run on the 5 highest-traffic screens plus every redesigned screen:

1. **Traversal order**: swipe right repeatedly through the whole screen. Focus must move in
   logical reading order (top-down, label-before-value); no focus traps, no unreachable controls,
   no focus landing on invisible/offscreen elements.
2. **Announcements**: every focused element announces role + label + state ("Button, Download,
   disabled"). Flag: raw resource names, "unlabeled", double announcements, decorative noise.
3. **Custom actions**: swipe/long-press-only gestures (dismiss, reorder, reveal) must be exposed
   as custom accessibility actions (Compose `Modifier.semantics { customActions = ... }`); verify
   via TalkBack menu (three-finger tap) → Actions.
4. **Dynamic changes**: trigger loading→content, errors, and snackbars; confirm they are
   announced (live region / `announceForAccessibility`), and dialogs move focus into themselves.
5. Complete one core user journey end-to-end with the screen off-limits to your eyes (screen
   curtain: TalkBack menu → settings) — if you cannot finish it, that is a Critical finding.

### 2.5 Font scale 200% survival

```powershell
adb shell settings put system font_scale 2.0
# restore: adb shell settings put system font_scale 1.0
```

Walk every screen at 2.0 (and spot-check 1.3): no clipped/ellipsized critical text, no
overlapping controls, no fixed-height containers cutting content, buttons grow rather than
truncate, scrolling appears where content no longer fits. Compose: the 200% preview annotation
required by `03-design/08-component-library.md` pre-screens components; this step verifies whole
screens. Common offenders: `maxLines = 1` on titles, dp-sized text (`fontSize = 14.dp`), bottom
bars with fixed height.

### 2.6 Keyboard and switch navigation

This check is the source for the **Keyboard / Switch** rows of the gate
(`07-checklists/accessibility.md`); run it on the top-5 traffic screens plus every redesigned
screen and record a per-screen pass/fail so the gate can cite this evidence.

Connect a hardware keyboard, or drive D-pad/Tab over adb:

```powershell
adb shell input keyevent KEYCODE_TAB        # advance focus
adb shell input keyevent KEYCODE_DPAD_DOWN  # directional focus
adb shell input keyevent KEYCODE_DPAD_CENTER # activate
```

```bash
adb shell input keyevent KEYCODE_TAB
adb shell input keyevent KEYCODE_DPAD_DOWN
adb shell input keyevent KEYCODE_DPAD_CENTER
```

Checklist hook — every item below must pass on each tested screen:

1. **Reachability**: Tab / D-pad reaches **every** interactive element; nothing is keyboard-only
   unreachable (no widget that can only be touched).
2. **Visible focus**: the focused element shows a focus indicator at **≥ 3:1 contrast** against
   its background (ties to the per-component `focused` state required by
   `03-design/08-component-library.md` §3.1a).
3. **Activation**: Enter / Space / D-pad-center activates the focused control.
4. **No traps**: focus never gets stuck in a sub-region (dialog, web view, custom control) with no
   keyboard way out.

This is the same traversal graph Switch Access uses, so a clean keyboard pass is the baseline
switch-nav check — a keyboard failure is automatically a switch-nav failure. Map results to the
gate's Keyboard/Switch rows; any failure is at least a High finding (Critical if a core journey
becomes impossible).

### 2.7 Reduced-motion survival

Verify the app honors the system "Remove animations" preference — the same reduced-motion contract
the motion pipeline defines (D14, `03-design/10-motion-design.md`). Enable it:

```powershell
adb shell settings put global animator_duration_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global window_animation_scale 0
# restore:
adb shell settings put global animator_duration_scale 1
adb shell settings put global transition_animation_scale 1
adb shell settings put global window_animation_scale 1
```

```bash
adb shell settings put global animator_duration_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global window_animation_scale 0
# restore:
adb shell settings put global animator_duration_scale 1
adb shell settings put global transition_animation_scale 1
adb shell settings put global window_animation_scale 1
```

Walk every redesigned screen plus the splash (`03-design/07-splash-redesign.md` §2.2). Survival
criteria: no essential information is conveyed **only** by motion; looping/auto-playing animations
stop or are removed; parallax, shimmer, and decorative motion fall back to a static state;
transitions degrade to an instant cut or simple fade; nothing becomes unusable or hidden because an
animation no longer runs. A screen that loses content or function under reduced motion is a High
finding (Critical if it blocks a core journey). Restore the three scales when done.

### 2.8 RTL mirroring pass

Verify the layout mirrors correctly for right-to-left locales even if no RTL locale ships today —
a future locale add must not require re-layout, and Android pseudo-locales expose mirroring bugs
cheaply. Toggle force-RTL (or a pseudo-locale):

```powershell
adb shell settings put global debug_force_rtl 1
# or pseudo-locale (also exposes truncation): adb shell setprop persist.sys.locale ar-XB
# restore:
adb shell settings put global debug_force_rtl 0
```

```bash
adb shell settings put global debug_force_rtl 1
# restore:
adb shell settings put global debug_force_rtl 0
```

Walk the top-5 + redesigned screens. Mirroring criteria: directional layout flips (lists, nav
rail/drawer, back/forward affordances, list-item chevrons); **directional icons** that imply
motion or direction are mirrored (use `android:autoMirrored="true"` / `Modifier` mirroring) while
non-directional glyphs (logos, media play) are **not**; start/end padding is used instead of
hardcoded left/right; text alignment follows the locale; no clipped or overlapping text from
mirrored constraints. Common offenders: hardcoded `paddingLeft/Right`, `Alignment.CenterStart`
assumed to mean "left", non-mirrored chevrons. Each defect is a finding (severity per Section 3;
a broken back affordance under RTL is High). Restore force-RTL when done.

---

## 3. Severity Mapping

Use the factory scale (Critical/High/Medium/Low, defined in `01-core/PROJECT_LIFECYCLE.md`) with
these anchors:

| Finding | Severity |
|---|---|
| Core journey impossible under TalkBack or at 200% font scale | Critical |
| Contrast failure on body text or primary actions | High |
| Missing label on icon-only action | High |
| Touch target < 48 dp on a frequent action | High |
| Contrast failure on secondary/decorative text | Medium |
| Illogical traversal order without functional loss | Medium |
| Touch target < 48 dp on rare/settings action | Medium |
| Decorative element announced (noise) | Low |
| Verbose but accurate announcement | Low |

## 4. Finding Format

Accessibility findings use their **OWN ID stream** `A11Y-NNN` (zero-padded 3, e.g. `A11Y-001`),
registered in `01-core/CONVENTIONS.md` per canonical decision D2. The old `A11Y-DES-xxx` format is
**abolished** — accessibility is a distinct stream from `DES-NNN` design findings, not a sub-series
of it. Example: `A11Y-014 — [High] Settings: "Clear cache" icon button unlabeled`. Each finding
records: screen (inventory ID), check (section number above, e.g. 2.3), evidence (screenshot path
under `docs/factory/reports/a11y/` or contrast pair `#fg/#bg = ratio`), severity (Critical/High/
Medium/Low), effort (S/M/L/XL), and fix recommendation. Findings land in
`docs/factory/audits/DESIGN_AUDIT.md` (audit phase) or the verification report (P8). The
pass/fail gate is `07-checklists/accessibility.md`, which requires **zero open Critical/High
findings**; this file generates the evidence that checklist consumes.

### Acceptance checklist

- [ ] Scanner captures saved for every inventoried screen, light and dark
- [ ] Compose/Espresso accessibility assertions added and green; lint a11y category triaged
- [ ] Contrast table covers all token pairs plus per-screen deviations
- [ ] Touch-target sweep done via Layout Inspector; all interactive nodes ≥ 48 dp
- [ ] Content-description greps run; every hit classified decorative vs informative
- [ ] TalkBack script executed on top-5 + redesigned screens; journey completed with screen curtain
- [ ] Keyboard/switch-nav pass (§2.6) run on top-5 + redesigned; reachability, visible focus, activation, no traps recorded per screen
- [ ] Reduced-motion survival (§2.7) walked with animation scales = 0; settings restored
- [ ] RTL mirroring pass (§2.8) walked with force-RTL; settings restored
- [ ] 200% font-scale walk completed; device settings restored (font scale, TalkBack off)
- [ ] All findings logged as `A11Y-NNN` (own stream, D2 — not `A11Y-DES-xxx`) with severity and evidence; zero Critical/High open
- [ ] Evidence handed to the gate `07-checklists/accessibility.md` (pass/fail authority)
