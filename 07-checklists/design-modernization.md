# Design Modernization Checklist

Gate for **P6 (Redesign Implementation) exit**. This checklist verifies that the redesign defined
in the DESIGN/IMPLEMENTATION handovers was implemented faithfully: Material 3 theme and tokens
applied, shared components used consistently, every IMPL ticket closed, visual evidence captured,
and — critically — that the work introduced **no behavior changes** (P6 is UI-only by contract).
P7 (ASO & Store Assets) must not begin until this checklist passes, because store screenshots
depend on the final UI.

> **Run when:** End of P6, after `01-core/prompts/09-ui-implementation.md` reports all tickets
> closed and `07-checklists/accessibility.md` has been run.
> **Run by:** The implementation agent (self-gate), then ideally a fresh agent against the
> IMPLEMENTATION handover for fidelity review.
> **Pass condition:** 100% of non-N/A items checked; every N/A carries a written justification.
> Record the completed copy as `docs/factory/reports/design-modernization-YYYY-MM-DD.md`.

All commands run from the target app repo root. "Modified screens" = every screen listed in the
IMPLEMENTATION handover's ticket table.

## Theme & Tokens

- [ ] Material 3 theme applied app-wide per `03-design/05-material3-modernization.md`: Compose
      screens use `MaterialTheme` with an M3 `ColorScheme`; remaining View-based screens inherit
      `Theme.Material3.*`. Verify: `rg "Theme.AppCompat|Theme.MaterialComponents" app/src/main/res`
      returns nothing for active themes; `rg "MaterialTheme" --type kotlin` covers entry points.
- [ ] Dark theme complete: every modified screen renders correctly in dark mode with a full dark
      `ColorScheme` (Compose) and/or `values-night` resources (Views). Verify:
      `adb shell cmd uimode night yes`, walk modified screens, then `adb shell cmd uimode night no`.
- [ ] Dynamic color decision recorded (adopted via `dynamicLightColorScheme()`/
      `dynamicDarkColorScheme()`, or explicitly declined) in `docs/factory/PROJECT_STATE.md`
      Decisions section. Verify: the decision entry exists with rationale.
- [ ] Zero hardcoded hex colors in modified screens — all color via theme tokens. Verify:
      `rg "Color\(0x" --type kotlin <modified screen files>` and
      `rg "#[0-9A-Fa-f]{6,8}" app/src/main/res/layout` (for touched layouts) return nothing,
      or each hit is a justified exception (e.g. brand asset) noted here.
- [ ] Typography and shape tokens come from the design-system extraction
      (`03-design/04-design-system-extraction.md` output): no ad-hoc `TextStyle(fontSize = ...)`
      in modified screens. Verify: `rg "fontSize\s*=" --type kotlin <modified screen files>` hits
      only theme definitions.

## Components

- [ ] Shared Empty / Loading / Error state components (per `03-design/08-component-library.md`)
      are used on **every** redesigned screen that can show those states — no per-screen
      re-implementations. Verify: `rg "EmptyState|LoadingState|ErrorState" --type kotlin` (use the
      project's actual component names from the handover) covers each screen in the ticket table.
- [ ] Component inventory in the DESIGN handover / `docs/factory/PROJECT_STATE.md` is updated
      with every shared component actually built (name, file path, screens using it). Verify:
      each listed file path exists (`Test-Path` each).

## Screens

- [ ] Every IMPL ticket in `docs/factory/handovers/IMPLEMENTATION_HANDOVER_v<N>.md` has
      Status = Done or Deferred (Deferred requires reason + owner note). Verify: scan the ticket
      table; no ticket is Not Started / In Progress / Blocked.
- [ ] Every COPY ticket (`COPY-NNN`) in the IMPLEMENTATION handover is implemented — the approved
      string change is in `app/src/main/res/values*/strings.xml`, and no un-ticketed string was
      changed. Verify: each `COPY-NNN` resolves to a strings.xml edit; cross-check
      `git diff <p6-start-tag>..HEAD -- "**/strings.xml"` against the COPY ticket list — any string
      change without a ticket is a FAIL.
- [ ] A redesigned screenshot is captured for every modified screen, including the dark variant,
      stored under `docs/factory/assets/screenshots/redesigned/`. Verify:
      `adb exec-out screencap -p > <name>.png` per screen; file count = 2 × screen count
      (light + dark), names match screen IDs (SCR-xx) from the screen inventory.
- [ ] Before/after pairs archived: each redesigned screenshot has its pre-P6 counterpart from
      `docs/factory/assets/screenshots/baseline/` (captured in P5 per
      `03-design/03-screenshot-capture.md`). Verify: matching SCR-xx filenames exist in both
      directories.
- [ ] Splash screen meets the acceptance criteria of `03-design/07-splash-redesign.md` (its
      dedicated ticket from prompt 09): correct `windowSplashScreen*` attributes / `SplashScreen`
      API usage, branded icon centered, no white-flash to first frame. Verify: cold-start the app,
      record a screen capture of the splash→first-frame transition, and check each acceptance bullet
      in `03-design/07-splash-redesign.md`.
- [ ] App icon meets the acceptance criteria of `03-design/06-icon-redesign.md` (mono stroke
      ~2dp; brief recorded in `docs/factory/reports/ICON_SPEC.md`): adaptive icon foreground/
      background present, themed-icon monochrome layer present. Verify: confirm
      `app/src/main/res/mipmap-anydpi*/ic_launcher*.xml` layers and check each acceptance bullet in
      `03-design/06-icon-redesign.md`; `Test-Path docs/factory/reports/ICON_SPEC.md`.

## Fidelity

- [ ] Per-screen spec conformance review done: each Done ticket's implementation compared against
      its DES-xx spec in the DESIGN handover (layout, spacing, components, states), with a
      one-line verdict per screen recorded in this report or the handover. Verify: the review
      table exists and has a row per Done ticket.
- [ ] Motion and state specs (transitions, loading shimmer, pressed states, etc.) implemented as
      specified, or explicitly Deferred with a note on the ticket. Verify: cross-check the
      IMPLEMENTATION handover's **§Motion Specification** (context | transition | duration |
      easing | reduced-motion fallback) and each ticket's `Motion:` field against the review
      table — every Motion line maps to an implemented behavior or an explicit Deferred note; no
      spec is silently dropped.

## Platform Behavior & Adaptivity

- [ ] Edge-to-edge and window insets correct on every modified screen — no content drawn under,
      or clipped by, the status/navigation bars or display cutout; gesture-nav and 3-button-nav
      both handled. Verify: `rg "enableEdgeToEdge|WindowInsets|safeDrawing|systemBars" --type kotlin`
      covers entry points; smoke each modified screen with gesture nav, then 3-button nav
      (`adb shell cmd overlay enable com.android.internal.systemui.navbar.threebutton`), confirming
      no overlap or clipping.
- [ ] Predictive back works (or is correctly opted in): no broken back behavior on Android 14+.
      Verify: `rg "android:enableOnBackInvokedCallback|OnBackInvokedCallback|PredictiveBack" --type kotlin --type xml`
      reflects the handover's decision; on a device with predictive-back developer option on, the
      back gesture shows the predictive animation and lands on the correct destination.
- [ ] Dynamic-color on/off matrix walked: app renders correctly with Material You dynamic color
      enabled AND with the static brand scheme, in both light and dark — matching the decision
      recorded above. Verify: toggle a system wallpaper/accent (or
      `adb shell settings put secure theme_customization_overlay_packages '...'`) to force a
      dynamic palette, walk modified screens, then disable; confirm no unreadable or off-brand
      surfaces in any of the four cells.
- [ ] Landscape / rotation smoke on every modified screen: rotate and confirm no crash, no lost
      state, no broken layout. Verify: `adb shell settings put system accelerometer_rotation 1`
      then rotate (`adb shell cmd window set-user-rotation 1` / back to `0`) through each modified
      screen; `adb logcat -d *:E | rg "FATAL"` empty.
- [ ] RTL correct on modified screens via the pseudo-locale: layouts mirror, no hardcoded
      left/right, no clipped mirrored text. Verify: enable
      `adb shell setprop debug.force_rtl 1` or set the RTL pseudo-locale
      (`adb shell am start -a android.settings.LOCALE_SETTINGS` → `ar-XB`), walk modified screens,
      screenshot, restore. Cross-check `rg "layout_marginLeft|layout_marginRight|paddingLeft|paddingRight" app/src/main/res/layout`
      against touched layouts (should be `Start`/`End`).
- [ ] Reduced-motion survival: with system animations off / "remove animations" on, the app
      remains usable and every motion spec falls back per its reduced-motion rule (no required
      transition becomes an instant dead-end or a blank frame). Verify:
      `adb shell settings put global animator_duration_scale 0` (and transition/window scales),
      walk modified screens, confirm reduced-motion fallbacks from the IMPLEMENTATION handover
      §Motion Specification engage; restore scales to `1`.

## Visual QA

- [ ] Visual-QA diffs completed per `03-design/12-design-qa.md`: redesigned captures compared
      against the REDESIGN proposal / handover reference frames, deviations triaged. Verify: the
      design-QA pass from `03-design/12-design-qa.md` was run and its diff/findings table is
      attached to this report or the handover, with every flagged deviation Resolved or Deferred
      with a note.

## Regression

- [ ] No behavior changes introduced — the P6 diff is UI-only. Verify: review
      `git diff <p6-start-tag>..HEAD --stat`; changes touch only UI layers (composables, layouts,
      themes, resources, navigation styling). Any ViewModel/repository/data change is flagged and
      justified here or the verdict is FAIL.
- [ ] Build verification Tier 2 per `02-engineering/07-build-verification.md` is green: clean
      assemble + unit tests + lint. Verify:
      `./gradlew clean :app:testDebugUnitTest :app:lintDebug :app:assembleDebug` exits 0.
- [ ] Smoke pass on the redesigned build: cold start plus every modified screen opened once, no
      crash. Verify: `adb logcat -d *:E | rg "FATAL"` returns nothing for the session.

## Documentation

- [ ] IMPLEMENTATION handover ticket table fully statused (mirrors the Screens group above) and
      the handover itself still passes `07-checklists/handover-validation.md` after edits.
- [ ] `docs/factory/CHANGELOG.md` updated: one dated entry per redesigned screen or ticket batch,
      plus an entry for this gate run. Verify: read the changelog.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Verdict | PASS / FAIL |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |

On PASS: advance the phase tracker in `docs/factory/PROJECT_STATE.md` to P7 and log this report in
`docs/factory/CHANGELOG.md`. On FAIL: fix, re-run failed groups in a new dated report.
