# Screenshot Capture Workflow

This document defines the operational workflow for capturing app screenshots: device setup, status
bar cleanup, capture commands, state variants, naming, and storage. It serves three consumers:
baseline evidence for the design audit (`01-core/prompts/05-design-audit.md`), after-state proof
during redesign implementation (`09-ui-implementation.md`), and raw material for store screenshots
(`11-screenshot-generation.md`). Captures that ignore this workflow (dirty status bars, random
naming, real user data) are rejected at handover validation.

## Step 1 — Emulator Selection & Setup

Capture on an emulator, not a personal device — reproducible resolution, clean state, no personal
data. Requirements:

| Property | Required value | Why |
|---|---|---|
| Device profile | Pixel-class AVD (e.g., Pixel 8) | Reference Material rendering, standard DPI |
| Resolution | 1080×2400 (~420 dpi, portrait) | Matches Play Store phone screenshot norms; consistent across projects |
| API level | Match the app's `targetSdk` (baseline: 36) | Audit what users on modern devices see |
| Image | Google Play image | Required for billing/paywall and dynamic color surfaces |

Create and boot the phone AVD (adjust system-image path to what `sdkmanager --list` shows as latest
stable):

```bash
sdkmanager "system-images;android-36;google_apis_playstore;x86_64"
avdmanager create avd -n factory-capture -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_8
emulator -avd factory-capture -no-snapshot-load
adb wait-for-device
```

```powershell
sdkmanager "system-images;android-36;google_apis_playstore;x86_64"
avdmanager create avd -n factory-capture -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_8
emulator -avd factory-capture -no-snapshot-load
adb wait-for-device
```

### Tablet / landscape profile (adaptive screens)

Any screen marked `responsive` / `tablet+` or orientation `landscape` / `both` in
`docs/factory/audits/SCREEN_INVENTORY.md` (`03-design/02-screen-inventory.md`) must also be captured
on a tablet-class AVD so the adaptive layout is auditable (`03-design/11-adaptive-layout.md`). Create
a second AVD; a Pixel Tablet profile gives an expanded window size class:

```bash
avdmanager create avd -n factory-capture-tablet -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_tablet
emulator -avd factory-capture-tablet -no-snapshot-load
adb wait-for-device
```

```powershell
avdmanager create avd -n factory-capture-tablet -k "system-images;android-36;google_apis_playstore;x86_64" -d pixel_tablet
emulator -avd factory-capture-tablet -no-snapshot-load
adb wait-for-device
```

Force orientation for landscape captures on either AVD (rotate, then re-navigate to the screen):

```bash
adb shell settings put system accelerometer_rotation 0
adb shell settings put system user_rotation 1   # 0=portrait, 1=landscape, 3=landscape-reverse
```

```powershell
adb shell settings put system accelerometer_rotation 0
adb shell settings put system user_rotation 1   # 0=portrait, 1=landscape, 3=landscape-reverse
```

Tag tablet/landscape captures with a `-tablet` / `-land` modifier after the theme segment (see
naming, Step 5) so they pair distinctly against phone-portrait baselines.

Verify exactly one device is attached before any capture session (`adb devices`); if several, add
`-s <serial>` to every adb command below. Re-enter demo mode (Step 2) after booting each AVD.

## Step 2 — Clean Status Bar (Demo Mode)

Run before every capture session. Demo mode pins the clock and fakes full battery/signal so
screenshots are diff-comparable across sessions. The demo clock is pinned to **10:00** everywhere
(`hhmm 1000`) — this is the canonical demo-mode setting in `01-core/CONVENTIONS.md` §Demo-mode
settings; never use any other time. The same adb commands work from bash or PowerShell (only the
binary-capture step in Step 3 differs by shell):

```bash
adb shell settings put global sysui_demo_allowed 1
adb shell am broadcast -a com.android.systemui.demo -e command enter
adb shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 1000   # 10:00, canonical
adb shell am broadcast -a com.android.systemui.demo -e command battery -e level 100 -e plugged false
adb shell am broadcast -a com.android.systemui.demo -e command network -e wifi show -e level 4
adb shell am broadcast -a com.android.systemui.demo -e command network -e mobile show -e datatype none -e level 4
adb shell am broadcast -a com.android.systemui.demo -e command notifications -e visible false
```

```powershell
adb shell settings put global sysui_demo_allowed 1
adb shell am broadcast -a com.android.systemui.demo -e command enter
adb shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 1000   # 10:00, canonical
adb shell am broadcast -a com.android.systemui.demo -e command battery -e level 100 -e plugged false
adb shell am broadcast -a com.android.systemui.demo -e command network -e wifi show -e level 4
adb shell am broadcast -a com.android.systemui.demo -e command network -e mobile show -e datatype none -e level 4
adb shell am broadcast -a com.android.systemui.demo -e command notifications -e visible false
```

Exit demo mode at the end of the session (either shell):

```bash
adb shell am broadcast -a com.android.systemui.demo -e command exit
```

Note: demo mode fakes the status bar **display** only; if a flow needs real network off (error
states), use `adb shell cmd connectivity airplane-mode enable` — the demo-mode wifi icon will
still show, which is acceptable for error-state evidence (record it in the capture log).

## Step 3 — Capture Commands

Canonical capture (bash / Git Bash / CMD):

```bash
adb exec-out screencap -p > SCR-001-default-light.png
```

PowerShell-safe variant — **never** use `>` with binary output in PowerShell 5.1 (it re-encodes as
UTF-16 and corrupts the PNG):

```powershell
adb exec-out screencap -p | Set-Content -AsByteStream SCR-001-default-light.png
```

If `Set-Content -AsByteStream` is unavailable (Windows PowerShell 5.1 lacks it), use the
device-side temp file pattern, which is always safe:

```powershell
adb shell screencap -p /sdcard/cap.png
adb pull /sdcard/cap.png SCR-001-default-light.png
adb shell rm /sdcard/cap.png
```

Validate every file after capture — a corrupt PNG fails silently otherwise:

```powershell
Get-Item SCR-001-default-light.png | Select-Object Length   # must be > 50 KB typically
```

and confirm the first bytes are the PNG magic (`89 50 4E 47`) if size looks suspicious.

For animations/transitions evidence (jank findings, splash screens), record video instead:

```powershell
adb shell screenrecord --time-limit 15 /sdcard/rec.mp4
adb pull /sdcard/rec.mp4 SCR-001-transition.mp4
```

## Step 4 — State & Theme Variants

### Dark mode

```powershell
adb shell cmd uimode night yes    # dark
adb shell cmd uimode night no     # light
```

Re-navigate to the screen after toggling (some activities recreate, some don't — if the screen
does NOT update, that itself is a finding for `03-design/01-design-audit.md`, H9).

### Font scale (accessibility captures)

```powershell
adb shell settings put system font_scale 1.0    # baseline
adb shell settings put system font_scale 1.5    # a11y variant — capture traffic-H screens
adb shell settings put system font_scale 2.0    # stress variant — only if 1.5 already breaks
adb shell settings put system font_scale 1.0    # ALWAYS reset when done
```

Name a11y variants with a `-fs15` / `-fs20` suffix after the theme segment.

### Display size (optional, for layout-robustness findings)

```powershell
adb shell wm density 540    # simulate larger display size
adb shell wm density reset
```

### Per-screen state staging

| State | How to stage |
|---|---|
| default | Demo content loaded (see staging rules below) |
| empty | `adb shell pm clear <package>`, complete onboarding, stop before adding content |
| loading | Throttle: emulator console `network speed gsm`, or capture within the load window; screenrecord and extract a frame if too fast |
| error | Connectivity present but request fails — block the API host or trigger a server/permission error; capture the failed-request UI |
| offline | `adb shell cmd connectivity airplane-mode enable`, then open the screen — captures the no-connectivity state, which is distinct from `error` (failed-while-connected). Disable with `... airplane-mode disable` after |
| dark | `cmd uimode night yes` + re-capture default state |

`offline` and `error` are separate states (D12): a screen may show a "you're offline" affordance for
no connectivity and a different "couldn't load" affordance for a failed request. Capture both when
the inventory marks the screen as doing network I/O.

## Step 5 — Naming Convention

```
SCR-<id>-<state>-<theme>[-<modifier>].png
```

- `<id>`: 3-digit SCR-ID from `docs/factory/audits/SCREEN_INVENTORY.md` (`03-design/02-screen-inventory.md`)
- `<state>`: `default` | `empty` | `loading` | `error` | `offline` (plus flow-specific states like `step2` when a screen has staged sub-states)
- `<theme>`: `light` | `dark`
- `<modifier>` (optional): `fs15`, `fs20`, `tablet`, `land`, `annotated`, `v2` (redesign iteration)

The canonical pattern is `SCR-<id>-<state>-<theme>.png` (per `01-core/CONVENTIONS.md`); modifiers
append after the theme segment. Examples: `SCR-003-empty-dark.png`, `SCR-007-default-light-fs15.png`,
`SCR-003-default-light-tablet.png`, `SCR-003-default-light-land.png`,
`SCR-004-error-light-annotated.png`. Lowercase except the SCR prefix; no spaces; PNG only for
stills.

## Step 6 — Directory Layout

All working captures live in the TARGET app repo under exactly two directories (paths per
`01-core/CONVENTIONS.md` §Canonical paths):

```
docs/factory/assets/screenshots/
├─ baseline/      # P5 audit evidence — current app, before any redesign; IMMUTABLE once audit ships
└─ redesigned/    # P6 after-states — same SCR-IDs/states/names as baseline for 1:1 visual diff
```

The spelling is `redesigned/` — never `redesign/`. There is **no** `screenshots/store/` directory:
store-ready frames live under `docs/factory/assets/store/<store>/<locale>/` and are generated from
`redesigned/` sources by prompt 11 (`04-aso/workflows/screenshot-optimization.md`), never directly
from raw captures.

Rules: `baseline/` is never modified after `docs/factory/audits/DESIGN_AUDIT.md` is finalized —
re-captures during P6 go to `redesigned/`. The `redesigned/` set must mirror baseline filenames
exactly (same SCR-IDs, states, theme, and `-tablet`/`-land` modifiers) so
`01-core/prompts/15-final-verification.md` can pair before/after by filename.

## Step 7 — App-State Staging Rules (demo content)

1. **Never expose real user data.** No real names, emails, phone numbers, account IDs, photos of
   people, or real documents. Always start from `adb shell pm clear <package>` and stage fresh
   demo content.
2. **Demo content must be plausible and category-appropriate**: realistic file names ("Q2 Invoice
   Draft.pdf"), believable counts (5–12 list items, not 1 and not 200), neutral images. Lorem
   ipsum is forbidden on traffic-H screens — it distorts hierarchy judgments and is unusable for
   store assets later.
3. **Stage to flatter nothing**: capture the screen as users actually meet it. Do not hide a
   banner ad, dismiss a known popup, or use a premium account to suppress the paywall unless that
   variant is captured *in addition to* the default experience.
4. **Deterministic where possible**: same demo data set across baseline and redesigned captures so
   visual diffs show design changes only. Record the staging steps per screen in a short capture
   log (`docs/factory/assets/screenshots/CAPTURE_LOG.md`): date, emulator, API level, app
   versionName, font scale, staging notes.
5. **Locale**: capture in the app's primary store locale; additional locales only when
   `04-aso/workflows/localization.md` is in scope.

## Session Checklist

- [ ] `adb devices` shows exactly the capture emulator
- [ ] Demo mode entered; clock pinned to **10:00**; battery 100; notifications hidden
- [ ] App data cleared and demo content staged per rules above
- [ ] For every SCR-ID: all states listed in the inventory's "States" column captured (incl.
      `offline` where the screen does network I/O); **dark captured for every inventoried screen**
- [ ] Adaptive screens (`responsive`/`tablet+`, `landscape`/`both` in the inventory) captured on the
      tablet AVD and/or in landscape, with `-tablet`/`-land` modifiers
- [ ] Traffic-H screens additionally captured at font scale 1.5
- [ ] Every PNG validated (size + opens correctly)
- [ ] Files named `SCR-<id>-<state>-<theme>[-<modifier>].png`, placed under `baseline/` or
      `redesigned/` (never `redesign/`, never a `screenshots/store/` dir)
- [ ] Capture log updated; font scale reset to 1.0; orientation reset; demo mode exited
