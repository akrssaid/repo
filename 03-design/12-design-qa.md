# Design QA / Visual-Regression Method

This document defines the design-QA method (canonical decision D12): how to pair every baseline
screenshot with its redesigned counterpart, diff them mechanically, triage each difference, sweep
the rendering-condition matrix, and feed the results into the P6 design-modernization gate. It is
consumed by `01-core/prompts/09-ui-implementation.md` (P6 — Redesign Implementation, end-of-phase
QA) and re-run during P8 via `01-core/prompts/15-final-verification.md`.

The point of design QA is not "did pixels change" — they did, that's the redesign — but **is every
change accounted for**: intentional changes map to a spec item, regressions get fixed, and nothing
drifts un-specced and unflagged into the release.

Conventions (paths, scales, IDs) live in `01-core/CONVENTIONS.md`; capture mechanics (demo mode,
10:00 clock, naming, directory layout) live in `03-design/03-screenshot-capture.md`. This file is
cited, it does not restate them.

---

## 1. Pairing Protocol (filename-mirror rule)

Baseline (P5) and redesigned (P6) captures live in the target repo per
`03-design/03-screenshot-capture.md`:

```
docs/factory/assets/screenshots/
├─ baseline/      # P5, immutable after the design audit ships
└─ redesigned/    # P6, MUST mirror baseline filenames exactly
```

Pairing is made executable by the **filename-mirror rule**: a redesigned screen is paired to its
baseline by **identical filename**, using the canonical `SCR-<id>-<state>-<theme>.png` convention.
`baseline/SCR-007-default-dark.png` pairs with `redesigned/SCR-007-default-dark.png` and nothing
else. Consequences:

- Every redesigned screen captures the **same SCR-IDs, states, and themes** as its baseline (the
  states contract per D12: default/empty/loading/error/offline, each in light and dark for
  inventoried screens). A redesigned file with no baseline twin is either a **new screen** (record
  it as such — no diff possible, spec-review only) or a **naming error** (fix the name).
- A baseline file with no redesigned twin is an **uncaptured screen** — the QA is incomplete until
  it exists or the screen is documented as intentionally removed.

Verify the pairing before diffing:

```bash
comm -3 \
  <(cd docs/factory/assets/screenshots/baseline && ls *.png | sort) \
  <(cd docs/factory/assets/screenshots/redesigned && ls *.png | sort)
```

```powershell
$base = Get-ChildItem docs\factory\assets\screenshots\baseline -Filter *.png | Select-Object -Expand Name
$redo = Get-ChildItem docs\factory\assets\screenshots\redesigned -Filter *.png | Select-Object -Expand Name
Compare-Object $base $redo   # SideIndicator => only in baseline; <= only in redesigned
```

Any unmatched filename is resolved (new screen, removed screen, or rename) before Section 2.

---

## 2. Scripted Pixel-Diff

Diff each pair with ImageMagick's `compare`, emitting a per-screen diff image plus an absolute
count of differing pixels (`AE` = Absolute Error metric).

**Install note.** Requires ImageMagick (`magick` / `compare` on PATH). Windows:
`winget install ImageMagick.ImageMagick`. macOS/Linux CI: `brew install imagemagick` /
`apt-get install imagemagick`. Verify with `magick -version`. Baseline and redesigned PNGs must be
the **same dimensions** (same AVD profile per `03-design/03-screenshot-capture.md`); mismatched
sizes make `AE` meaningless — re-capture on the reference AVD.

Single pair:

```bash
magick compare -metric AE \
  docs/factory/assets/screenshots/baseline/SCR-007-default-dark.png \
  docs/factory/assets/screenshots/redesigned/SCR-007-default-dark.png \
  docs/factory/reports/design-qa/diff/SCR-007-default-dark-diff.png
# prints the differing-pixel count to stderr; the diff image highlights changed pixels in red
```

```powershell
magick compare -metric AE `
  docs/factory/assets/screenshots/baseline/SCR-007-default-dark.png `
  docs/factory/assets/screenshots/redesigned/SCR-007-default-dark.png `
  docs/factory/reports/design-qa/diff/SCR-007-default-dark-diff.png
```

Sweep every pair, writing one diff image per screen into `docs/factory/reports/design-qa/diff/`:

```bash
mkdir -p docs/factory/reports/design-qa/diff
for f in docs/factory/assets/screenshots/redesigned/*.png; do
  name=$(basename "$f")
  base="docs/factory/assets/screenshots/baseline/$name"
  [ -f "$base" ] || { echo "NO BASELINE: $name"; continue; }
  ae=$(magick compare -metric AE "$base" "$f" \
        "docs/factory/reports/design-qa/diff/${name%.png}-diff.png" 2>&1)
  echo "$name  AE=$ae"
done
```

```powershell
New-Item -ItemType Directory -Force docs\factory\reports\design-qa\diff | Out-Null
Get-ChildItem docs\factory\assets\screenshots\redesigned -Filter *.png | ForEach-Object {
    $name = $_.Name
    $base = "docs\factory\assets\screenshots\baseline\$name"
    if (-not (Test-Path $base)) { Write-Host "NO BASELINE: $name"; return }
    $out  = "docs\factory\reports\design-qa\diff\$($_.BaseName)-diff.png"
    $ae   = (magick compare -metric AE $base $_.FullName $out) 2>&1
    Write-Host "$name  AE=$ae"
}
```

The `AE` count and the diff image are **triage inputs, not a pass/fail**: a high count on a fully
restyled screen is expected. Anti-aliasing and sub-pixel text rendering produce small non-zero
counts even on "unchanged" regions; use the diff image (where the difference *is*) to judge, not
the raw number. Use deterministic demo content (same staged data both runs, per
`03-design/03-screenshot-capture.md` §7) so diffs show design changes only.

---

## 3. Triage Rubric & Disposition

Open each diff image and classify **every** highlighted region into one of three dispositions:

| Disposition | Definition | Action |
|---|---|---|
| **Intentional** | The change maps to a specific spec item — a DES finding fix, an IMPL ticket, a token change, or a stated component/motion/adaptive spec | Record the mapping (DES-/IMPL-/token id). Approved. |
| **Regression** | An unintended change — clipped text, lost padding, wrong token, broken alignment, a control that moved or disappeared without a spec | File a finding (DES-/A11Y-NNN as appropriate) and fix before gate PASS. |
| **Un-specced drift** | A real change that is neither a regression nor traceable to any spec — "it looks a bit different and nobody asked for it" | Must be **flagged**: either retro-spec it (add the spec item and reclassify Intentional) or revert it. It may not ship silently. |

Fill the disposition table — one row per **changed region**, not per screen (a screen may have
several):

| Screen (SCR-id-state-theme) | Region | AE | Disposition | Maps to / finding | Resolution |
|---|---|---|---|---|---|
| SCR-007-default-dark | header bar color | 4120 | Intentional | IMPL-002 (M3 surfaceContainer) | approved |
| SCR-007-default-dark | body line 3 clipped | 80 | Regression | DES-031 | fix in IMPL-009 |
| SCR-012-default-light | FAB shadow softer | 240 | Un-specced drift | — | revert to spec elevation |

This table is the **Fidelity** evidence for the P6 gate
(`07-checklists/design-modernization.md` → Fidelity group): the gate's "no spec is silently
dropped" check reads it directly. The hard rule — **no un-specced drift ships unflagged** — is a
gate failure if violated.

---

## 4. Cross-Matrix Sweep

Pixel-diffing covers the default rendering condition. The cross-matrix sweep verifies the redesign
survives the **rendering conditions users actually hit** — independent of the baseline pairing,
because the baseline app may never have supported them. Sweep, per screen:

**light/dark × font scale 1.0/1.5 × dynamic color on/off** → 8 cells per screen.

Stage each axis (commands per `03-design/03-screenshot-capture.md`):

```bash
adb shell cmd uimode night yes            # / no       (theme)
adb shell settings put system font_scale 1.5   # / 1.0   (font scale)
# dynamic color: set a vivid wallpaper / toggle "Wallpaper colors" theme on the device,
# then relaunch — dynamic schemes derive from wallpaper on Android 12+
adb shell settings put system font_scale 1.0   # ALWAYS reset
```

```powershell
adb shell cmd uimode night yes
adb shell settings put system font_scale 1.5
adb shell settings put system font_scale 1.0   # ALWAYS reset
```

Sign off each cell in a per-screen table (PASS = renders correctly: no clipped/overlapping text,
no token that breaks under dynamic color, contrast holds, layout intact):

| Screen | L·1.0·dyn-off | L·1.0·dyn-on | L·1.5·dyn-off | L·1.5·dyn-on | D·1.0·dyn-off | D·1.0·dyn-on | D·1.5·dyn-off | D·1.5·dyn-on |
|---|---|---|---|---|---|---|---|---|
| SCR-007 | PASS | PASS | PASS | PASS | PASS | PASS | FAIL | PASS |

A FAIL cell becomes a finding (font-scale clipping → likely an `03-design/09-accessibility-review.md`
A11Y finding; dynamic-color breakage → a DES finding). Font scale 1.5 is the canonical a11y
capture variant; escalate to 2.0 only where 1.5 already breaks (matches the capture workflow).
Dynamic color is swept only when the app adopts it — if dynamic color was explicitly declined
(recorded in PROJECT_STATE Decisions), collapse those columns to N/A.

---

## 5. Recording Into Ticket Status & the Gate

Design QA does not have its own gate; it produces the evidence the **P6 design-modernization gate**
consumes and writes back into ticket status:

1. **Per-ticket QA verdict.** For each IMPL ticket's screens, the disposition table (Section 3)
   and the matrix sweep (Section 4) yield a one-line QA verdict. A ticket with an open Regression
   or un-specced drift on its screens is **not Done** — its status in
   `docs/factory/PROJECT_STATE.md` stays In Progress until resolved.
2. **Gate Fidelity section.** The disposition table is the artifact the
   `07-checklists/design-modernization.md` Fidelity items check: every Done ticket's screens
   compared to spec, no silently dropped spec, no un-specced drift. Attach this report (saved as
   `docs/factory/reports/design-qa/design-qa-YYYY-MM-DD.md`) to the gate run.
3. **Gate PASS condition addition.** The gate may PASS only when, for every redesigned screen,
   a diff image and a disposition exist, and no row is left as un-specced drift without a
   flag/resolution. Record the QA verdict (PASS / PASS with waivers / FAIL — per the D3 scale; the
   word "CONDITIONAL" is abolished) in the report and in `docs/factory/CHANGELOG.md`.

---

## Acceptance

- [ ] Every redesigned screen is paired to a baseline by the filename-mirror rule, or documented as a new/removed screen
- [ ] Pixel-diff run for every pair; one diff image per screen under `docs/factory/reports/design-qa/diff/`
- [ ] Every changed region triaged Intentional / Regression / Un-specced drift in the disposition table
- [ ] Every Intentional region maps to a DES/IMPL/token/spec id; every Regression has a finding + fix
- [ ] **No un-specced drift ships unflagged** — each such region retro-specced or reverted
- [ ] Cross-matrix sweep (light/dark × font 1.0/1.5 × dynamic color on/off) signed off per screen; every FAIL is a finding
- [ ] Results written back to ticket status and attached to the P6 design-modernization gate's Fidelity section
- [ ] Report saved as `docs/factory/reports/design-qa/design-qa-YYYY-MM-DD.md` and logged in `CHANGELOG.md`
