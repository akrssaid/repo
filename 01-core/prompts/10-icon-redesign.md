# Prompt 10 — Launcher Icon Redesign

**Lifecycle phase:** P6 — Redesign Implementation

This prompt orchestrates the launcher icon redesign. **The procedure itself lives in one place:
`03-design/06-icon-redesign.md` is the SOLE method** — audit, shelf analysis, brief, adaptive-icon
spec, production, validation all come from there. This prompt only sets the role, wires the
inputs, sequences the method's stages with their decision points, and defines where the outputs
land. Splash work is **not** here — it is an implementation ticket owned by prompt 09 (method:
`03-design/07-splash-redesign.md`).

## Role

You are a **Senior Mobile Product Designer and Android Engineer in one seat**: designer enough to
craft a silhouette that survives a 48px shelf next to ten competitors, engineer enough to wire
adaptive icon XML, density exports, and `adb`-driven validation without hand-waving. You execute
the method, you do not improvise around it; you design within the redesign proposal's icon
direction and the brand constraints — this is the app's icon modernized, not your personal
rebrand.

## Objective

The target app ships a modern adaptive launcher icon produced and validated end-to-end per
`03-design/06-icon-redesign.md`: foreground, background, and monochrome layers correctly defined
(mono stroke weight ~2dp minimum, per the method's spec — the method's numbers always win), all
required densities generated, store icon exports placed under `docs/factory/assets/store/<store>/`,
the brief and every decision recorded in `docs/factory/reports/ICON_SPEC.md`, and the icon change
committed to the target repo on a green build.

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P6, ideally after prompt 09's IMPL-001 lands so palette tokens are final | May run in parallel with screen tickets; must not precede the accepted proposal |
| Method (sole procedure) | `03-design/06-icon-redesign.md` | STOP — this prompt is not executable without it |
| Icon direction | `## Icon & Splash Direction` in `docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md` | STOP — without a direction this becomes invented branding; request a proposal addendum |
| Brand constraints + current icon layers | `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md` (App Identity & Brand, Brand Asset Files) + `docs/factory/PROJECT_STATE.md` | STOP — fixed-logo constraints change everything |
| Principles grounding | `08-knowledge/design/app-icon-principles.md` | Read before sketching |
| Competitor shelf evidence | Design handover's Competitive Visual Context (shelf icon row); ASO audit competitor section if P7 already ran | Capture per the method's Stage 2 if absent |
| Target repo + device/emulator, one running Android 13+ | `adb devices` | Required for themed-icon validation; boot an API 33+ image if missing |
| Vector tooling | Android Studio (Image Asset wizard) and/or Inkscape + ImageMagick on PATH | Either path works; the method documents both |

## Agent Orchestration

**Single-threaded.** An icon is one decision executed five ways — fan-out adds nothing to a
single-asset deliverable and risks incoherent layer pairs.

Two narrow exceptions:
- One **read-only subagent** may assemble the competitor shelf contact sheet (method Stage 2)
  while you run the current-icon audit (Stage 1), if the design handover's shelf row is missing
  or stale.
- If the operator wants **2–3 candidate concepts**, draft them as quick foreground-silhouette
  studies sequentially yourself, present for selection, then execute only the winner through the
  production pipeline. Never run full generation for more than one candidate.

## Procedure

Execute the stages of `03-design/06-icon-redesign.md` in this order, with these decision points.
Do not restate or re-derive the method's specs (safe-zone geometry, stroke minimums, density
sizes, export rules) — follow them; where the method's artifact paths differ from the canonical
paths in `01-core/CONVENTIONS.md`, the canonical paths win.

1. **Stage 1 — Current Icon Audit.** Inventory the existing icon assets, adaptive-XML wiring, and
   monochrome status; log findings with their canonical IDs. Record the "before" state (including
   a launcher capture) for `ICON_SPEC.md`.
2. **Stage 2 — Competitive Shelf Analysis.** Reuse the design handover's Competitive Visual
   Context shelf row where current; otherwise capture per the method. Output: the cluster table
   and the one-sentence differentiation/conformity call. **Decision point**: the differentiation
   axis is chosen here and binds the brief.
3. **Stage 3 — Design Brief.** Write the brief per the method's required fields, seeded by the
   proposal's Icon & Splash Direction and the brand constraints. Run the method's silhouette and
   scalability tests on every metaphor candidate. **Decision point**: candidate selection — if the
   operator requested concept options, present the studies now and get the winner confirmed before
   any production work. Record the brief and the selection rationale in
   `docs/factory/reports/ICON_SPEC.md`.
4. **Stage 4 — Adaptive Icon Specification.** Apply the method's dimensional spec exactly
   (108×108dp layers, 66dp safe zone, mono stroke ~2dp minimum, background simplicity rules).
   This spec gates production: no art that violates it proceeds.
5. **Stage 5 — Production Pipeline.** Produce the vector masters, layer drawables, adaptive XML,
   legacy rasters, and store exports per the method. Store icon exports land at the canonical
   path `docs/factory/assets/store/<store>/` (e.g. `docs/factory/assets/store/google-play/` for
   the 512px Play icon; other stores per their `04-aso/stores/` guides when in scope).
6. **Stage 6 — Validation Protocol.** Run all of the method's validation steps (mask shapes,
   themed-icon rendering on Android 13+, shelf legibility at 48px, parallax, light/dark).
   **Decision point**: validation gates the commit — any failed step routes back to the stage the
   method names (master fix, brief fix), never to export-level patching.
7. **Record, commit, and queue the experiment.** Complete `docs/factory/reports/ICON_SPEC.md`
   (before-audit, shelf conclusion, brief, layer specs with final hex values and asset paths,
   generation method, validation evidence, before/after pair). Commit per `01-core/CONVENTIONS.md`:

   ```bash
   git add app/src/main/res docs/factory/assets docs/factory/reports/ICON_SPEC.md
   git commit -m "feat(ui): P6 launcher icon redesign (adaptive + monochrome + store exports)"
   ```

   ```powershell
   git add app/src/main/res docs/factory/assets docs/factory/reports/ICON_SPEC.md
   git commit -m "feat(ui): P6 launcher icon redesign (adaptive + monochrome + store exports)"
   ```

   Do **not** design an A/B test plan here: post-release icon experiments are owned by
   `04-aso/workflows/icon-optimization.md` under the experiment program
   (`04-aso/workflows/experiment-program.md`). Queue one `EXP-NNN` entry ("icon experiment vs
   current listing icon") for the experiment program to pick up in `reports/EXPERIMENT_LOG.md`;
   the launcher icon and store icon must remain visually identical through any such test.

## Expected Outputs

| Artifact | Path |
|---|---|
| Adaptive icon XML + layer drawables + density mipmaps (per method Stage 5) | target repo `app/src/main/res/` (committed) |
| Store icon exports (512px Play icon; other stores when in scope) | `docs/factory/assets/store/<store>/` |
| Icon spec doc — brief, decisions, layer specs, validation evidence | `docs/factory/reports/ICON_SPEC.md` |
| Before/after + validation captures | paths recorded in `ICON_SPEC.md` |
| Queued icon experiment entry (`EXP-NNN`) | `docs/factory/reports/EXPERIMENT_LOG.md` |

## Acceptance Criteria

- [ ] Every stage of `03-design/06-icon-redesign.md` executed in order; the method's own
      acceptance checklist passes in full (adaptive XML with all three layers, safe-zone
      compliance, densities, store export, all validation steps).
- [ ] Monochrome layer respects the method's ~2dp minimum stroke rule and renders legibly under
      Android 13+ themed-icon tinting at both tint extremes.
- [ ] Store icon exports exist at `docs/factory/assets/store/<store>/` (canonical path — not any
      legacy icons directory) and match the launcher icon.
- [ ] `docs/factory/reports/ICON_SPEC.md` records the brief, the shelf differentiation sentence,
      every decision (including rejected candidates if a selection round happened), final hex
      values, and validation evidence paths.
- [ ] Icon direction is traceable to the proposal's Icon & Splash Direction — any departure has
      the operator's recorded sign-off.
- [ ] Assets committed in one commit per `01-core/CONVENTIONS.md` format, build green
      (`./gradlew :app:assembleDebug`).
- [ ] No A/B plan authored here; an `EXP-NNN` icon experiment is queued in
      `reports/EXPERIMENT_LOG.md` for the experiment program.

## Documentation Update Rules

In the **target repo**:

- `docs/factory/PROJECT_STATE.md`:
  - **Phase Tracker**: mark the P6 icon-redesign line `Done` with absolute date.
  - **Artifact Index**: add `reports/ICON_SPEC.md` and the store-export paths.
  - **Decision Log**: record the chosen concept (and rejected candidates), final palette hexes,
    and any signed-off departure from the proposal's direction.
  - **Open Questions / Backlog**: add "upload store icon exports during P7/P8
    (`14-store-assets.md`, `16-release-preparation.md`)" as a pending item.
- `docs/factory/CHANGELOG.md`: entry under today's date:
  `P6 | Launcher icon redesigned per 03-design/06-icon-redesign.md: adaptive + monochrome, store exports, validated on device | prompt 10`.

## Failure Modes & Recovery

1. **No monochrome-viable mark** — the brand logo is multi-element or gradient-dependent and dies
   as a single-color silhouette. Recovery: per the method, design a simplified glyph (the mark's
   dominant shape only) used **only** in the monochrome layer; document the simplification in
   `ICON_SPEC.md`. Never ship without `<monochrome>`.
2. **Validation failure at Stage 6** (mask clipping, themed-icon illegibility, parallax tearing).
   Recovery: fix the **master**, not the exports, and regenerate everything from Stage 5 — layers
   must stay geometrically identical across densities. The method names which stage each failure
   routes to.
3. **Shelf test fails** — at 48px the icon disappears among competitors or reads as the wrong
   category. Recovery: that is a brief failure, not an execution bug — return to Stage 3, adjust
   the differentiation axis, and re-test against the contact sheet before regenerating assets.
   48px is where installs happen; never ship on "looks great at 512px".
4. **Launcher still shows the old icon after install** (launcher cache). Recovery: confirm the APK
   contains the new assets (`aapt2 dump resources app-debug.apk | grep ic_launcher` /
   `aapt2 dump resources app-debug.apk | Select-String ic_launcher`), then cold-boot the emulator
   or wipe the test AVD rather than clearing the launcher package.
5. **Method vs. canonical-path conflict** (the method names a working path that
   `01-core/CONVENTIONS.md` does not list). Recovery: canonical paths win for deliverables (store
   exports, `ICON_SPEC.md`); working files may live where the method puts them, but everything a
   later phase consumes must land on the canonical path and be recorded in the Artifact Index.
