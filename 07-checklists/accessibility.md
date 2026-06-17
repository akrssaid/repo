# Accessibility Checklist

Gate run **within P6 (Redesign Implementation) and again within P8 (Verification & Release)**.
This checklist verifies the app meets the factory's accessibility floor — WCAG-derived contrast,
touch target sizes, labeling, font scaling, and screen-reader usability — using the procedures
defined in `03-design/09-accessibility-review.md`. It is invoked by that guide during P6 and by
`01-core/prompts/15-final-verification.md` during P8; the P8 run must be on the release candidate
build. Accessibility regressions are user-facing defects and Play vitals/review risks: a FAIL here
blocks both the P6 exit gate (`07-checklists/design-modernization.md` expects this to have run)
and release sign-off.

> **Run when:** During P6 after UI implementation, and during P8 on the release candidate.
> **Run by:** The agent executing `03-design/09-accessibility-review.md` (P6) or
> `01-core/prompts/15-final-verification.md` (P8).
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification.
> Record the completed copy as `docs/factory/reports/accessibility-YYYY-MM-DD.md`.

"Key screens" = the top-5 traffic screens identified in the screen inventory
(`docs/factory/audits/` per `03-design/02-screen-inventory.md`). Sample each item on key screens
at minimum; full coverage where stated.

Defects found here are logged as accessibility findings on their own stream using the `A11Y-NNN`
ID format (e.g. `A11Y-001`) per `01-core/CONVENTIONS.md` — the `A11Y-DES-xxx` format is abolished.

## Contrast

- [ ] Body text contrast ≥ 4.5:1 against its background, sampled on every key screen, **light
      theme**. Verify per `03-design/09-accessibility-review.md`: eyedropper the rendered
      screenshot (foreground + background hex) and compute ratio with a WCAG contrast formula;
      record screen → ratio pairs here.
- [ ] Body text contrast ≥ 4.5:1, same sampling, **dark theme**. Verify: switch with
      `adb shell cmd uimode night yes`, re-sample, record pairs; restore with
      `adb shell cmd uimode night no`.
- [ ] Large text (≥ 18sp regular / 14sp bold) and meaningful UI components (icons, input borders,
      focus indicators) ≥ 3:1, both themes, sampled per key screen. Verify: same eyedropper
      method; record the lowest ratio found per screen.
- [ ] Focus indicator has ≥ 3:1 contrast against the adjacent (unfocused) background, so the
      keyboard/D-pad focus ring is perceivable on every key screen, both themes. Verify: move
      focus with a hardware/virtual keyboard or D-pad onto a focusable element, screenshot, then
      eyedropper the focus-ring color against the surface immediately outside it and compute the
      ratio with a WCAG formula; record screen → ratio. (Distinct from the large-text 3:1 item
      above — this is specifically the focus-state indicator.)
- [ ] No information conveyed by color alone (e.g. error state = red only). Verify: review key
      screens for a secondary cue (icon, text, underline) on every color-coded state.

## Touch

- [ ] All interactive targets ≥ 48dp × 48dp on every key screen. Verify: enable
      `adb shell settings put global debug_layout 1` (Show layout bounds) or use Accessibility
      Scanner; for Compose, confirm `minimumInteractiveComponentSize` is not disabled
      (`rg "LocalMinimumInteractiveComponentSize|minimumInteractiveComponentSize" --type kotlin`).
      Record any sub-48dp target and its fix or N/A justification. Disable debug overlay after:
      `adb shell settings put global debug_layout 0`.
- [ ] Adjacent targets have ≥ 8dp spacing or distinct bounds (no accidental-tap clusters).
      Verify: visual pass with layout bounds enabled on toolbars, list item action rows, and
      bottom bars.

## Labels

- [ ] Every icon-only actionable element (IconButton, FAB, toolbar action, ImageView with click)
      has a meaningful `contentDescription`. Verify:
      `rg "contentDescription\s*=\s*null" --type kotlin` — every hit must be a genuinely
      decorative element; cross-check actionables via Accessibility Scanner or a TalkBack sweep.
- [ ] Decorative images are excluded from the accessibility tree (`contentDescription = null` in
      Compose; `importantForAccessibility="no"` in XML). Verify: spot-check decoratives on key
      screens with TalkBack — they must not receive focus.
- [ ] Every form field has a programmatic label (Compose `label`/semantics; XML
      `android:hint`/`labelFor`). Verify: TalkBack focus on each field in key forms announces the
      field's purpose, not "edit box".
- [ ] Dynamic content changes (snackbars, validation errors, loading completion) are announced.
      Verify: trigger each on key screens under TalkBack; announcement occurs (live region /
      `announceForAccessibility` per `03-design/09-accessibility-review.md`).

## Scaling

- [ ] App is usable at font scale 2.0: no clipped, truncated-without-ellipsis, or overlapping
      text on key screens; all actions still reachable. Verify:
      `adb shell settings put system font_scale 2.0`, walk key screens, screenshot each, then
      restore `adb shell settings put system font_scale 1.0`. Archive screenshots under
      `docs/factory/assets/screenshots/a11y/`.
- [ ] Text uses `sp` (not `dp`) everywhere; no fixed-height containers clipping scaled text.
      Verify: `rg "textSize=\"[0-9]+dp\"" app/src/main/res` returns nothing; Compose text styles
      use `.sp`.
- [ ] Display size at largest setting does not break key screens. Verify:
      `adb shell wm density 540` (on a ~420dpi device), walk key screens, restore with
      `adb shell wm density reset`.

## Screen Reader

- [ ] TalkBack pass completed on all top-5 traffic screens. Enable:
      `adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService`
      (disable by clearing the setting after the pass).
- [ ] Focus/reading order is sensible on each of the 5 screens (top-to-bottom, logical grouping,
      no focus traps). Verify: swipe-navigate the full screen; record any out-of-order element.
- [ ] Announcements are meaningful: no raw resource names, no "unlabeled", state included where
      relevant ("selected", "expanded"). Verify: listen during the pass; record defects.
- [ ] Custom actions (swipe-to-delete, drag handles, long-press menus) are reachable via TalkBack
      custom actions menu or an accessible alternative. Verify: invoke each gesture-driven action
      through TalkBack on screens that have them, or mark N/A with justification.

## Input & Navigation

- [ ] Keyboard and switch-access navigation work on key screens per
      `03-design/09-accessibility-review.md` §2.6: every interactive element is reachable and
      operable without touch, focus order is logical, and there are no focus traps. Verify: attach
      a hardware/Bluetooth keyboard (or enable Switch Access) and Tab/arrow through each key screen,
      activating controls with Enter/Space; or use the on-screen keyboard-nav method in §2.6.
      Record any element that cannot be reached or activated as an `A11Y-NNN` finding.

## Motion & Layout

- [ ] Reduced-motion survival: with system animations removed / "remove animations" enabled, key
      screens stay usable and no essential transition collapses into a blank frame or dead-end.
      Verify: `adb shell settings put global animator_duration_scale 0` (plus transition and window
      animation scales), walk key screens, confirm reduced-motion fallbacks engage; restore all
      scales to `1`.
- [ ] RTL mirroring correct on key screens: layouts mirror, directional icons/gestures flip where
      appropriate, no clipped or overlapping mirrored text. Verify: set the RTL pseudo-locale
      `ar-XB` (or `adb shell setprop debug.force_rtl 1`), walk key screens, screenshot, restore;
      cross-check `rg "Left|Right" app/src/main/res/layout` for hardcoded directions (should be
      `Start`/`End`).

## Lint

- [ ] All accessibility lint warnings triaged: `ContentDescription`, `TouchTargetSizeCheck`,
      `KeyboardInaccessibleWidget`, `ClickableViewAccessibility`, `LabelFor`. Verify:
      `./gradlew :app:lintRelease`, open the lint report
      (`app/build/reports/lint-results-release.html`), record count per issue ID and the
      disposition (fixed / baselined with reason) for each.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / FAIL |
| Phase of run | P6 / P8 |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |

Log the report in `docs/factory/CHANGELOG.md`. A P6 PASS does not waive the P8 run: re-run on the
release candidate, since theming, copy, and dependency changes after P6 can regress accessibility.
