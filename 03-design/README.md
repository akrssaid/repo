# 03-design — Design Method Library

This directory holds the factory's design methods: concrete, tool-level procedures for auditing,
documenting, and modernizing the UI of any target Android app. The prompts in `01-core/prompts/`
(primarily 05–11) reference these docs at execution time; the prompts say *when* and *who*, these
docs say *exactly how*. All outputs land in the target app's repo under `docs/factory/`.

All conventions — the ID registry (SCR/DES/SYS/A11Y/COPY/…), scales (severity/effort/status),
naming, canonical paths, and the artifact-contract matrix — are defined ONCE in
`01-core/CONVENTIONS.md`. These method docs cite it rather than restating it; where this README or a
method appears to define a rubric, ID format, or path, `01-core/CONVENTIONS.md` is authoritative on
any conflict.

## Canonical paths for the design system

All paths are repo-relative to the target app's `docs/factory/` (full matrix in
`01-core/CONVENTIONS.md` §Canonical paths). The design-system pipeline uses exactly these:

| Artifact | Canonical path | Produced by |
|---|---|---|
| Screen inventory | `audits/SCREEN_INVENTORY.md` (separate file, never embedded) | `02-screen-inventory.md` |
| Design audit (incl. embedded design-system extraction) | `audits/DESIGN_AUDIT.md` | `01-design-audit.md` + `04-design-system-extraction.md` |
| Baseline screenshots | `assets/screenshots/baseline/` | `03-screenshot-capture.md` |
| Redesigned screenshots | `assets/screenshots/redesigned/` (spelling: `redesigned`, never `redesign`) | `03-screenshot-capture.md`, prompt 09 |
| Store-ready frames | `assets/store/<store>/<locale>/` | prompt 11 |
| Capture log | `assets/screenshots/CAPTURE_LOG.md` | `03-screenshot-capture.md` |

The "Current Design System" census is **embedded inside** `audits/DESIGN_AUDIT.md` (a section of the
design-audit report), not a standalone file. Screenshot naming is `SCR-<id>-<state>-<theme>.png`;
the demo-mode clock is pinned to **10:00** in every capture.

## Method Docs

| Doc | Method | Consumed by (prompts) | Lifecycle phase |
|---|---|---|---|
| `03-design/01-design-audit.md` | Heuristic UX/design audit with scoring rubric, evidence rules, DES-xxx findings | `01-core/prompts/05-design-audit.md` | P5 |
| `03-design/02-screen-inventory.md` | Complete screen inventory: static + dynamic discovery, SCR-ID records, coverage check | `01-core/prompts/05-design-audit.md`, `06-design-handover.md` | P5 |
| `03-design/03-screenshot-capture.md` | Operational screenshot capture: emulator setup, demo mode, naming, directory layout | `01-core/prompts/05-design-audit.md`, `09-ui-implementation.md`, `11-screenshot-generation.md` | P5, P6 |
| `03-design/04-design-system-extraction.md` | Extract the CURRENT design system from code: color/type/spacing census + divergence report | `01-core/prompts/05-design-audit.md`, `06-design-handover.md` | P5 |
| `03-design/05-material3-modernization.md` | Material 3 migration playbook: theme migration, dynamic color, component priority order | `01-core/prompts/07-design-redesign.md`, `08-implementation-handover.md`, `09-ui-implementation.md` | P5, P6 |
| `03-design/06-icon-redesign.md` | App icon redesign: adaptive icon spec, monochrome layer, store icon variants | `01-core/prompts/10-icon-redesign.md` | P6 |
| `03-design/07-splash-redesign.md` | Splash screen modernization via `androidx.core:core-splashscreen` | `01-core/prompts/09-ui-implementation.md` | P6 |
| `03-design/08-component-library.md` | Building the target app's reusable component layer during redesign | `01-core/prompts/07-design-redesign.md`, `09-ui-implementation.md` | P5, P6 |
| `03-design/09-accessibility-review.md` | Accessibility review: TalkBack pass, contrast, touch targets, font scaling | `01-core/prompts/05-design-audit.md`, `15-final-verification.md` | P5, P8 |
| `03-design/10-motion-design.md` | Motion design method: M3 motion tokens, transition decision table, predictive-back, reduced-motion contract, handover flow | `01-core/prompts/07-design-redesign.md`, `08-implementation-handover.md`, `09-ui-implementation.md` | P5, P6 |
| `03-design/11-adaptive-layout.md` | Adaptive layout method: window size classes, tablet/foldable/orientation, list-detail & canonical layouts | `01-core/prompts/07-design-redesign.md`, `08-implementation-handover.md`, `09-ui-implementation.md` | P5, P6 |
| `03-design/12-design-qa.md` | Design QA method: post-implementation visual/interaction/a11y/motion verification against the handover | `01-core/prompts/09-ui-implementation.md`, `15-final-verification.md` | P6, P8 |

## How the methods chain in P5 → P6

1. **Inventory first** — `02-screen-inventory.md` produces the SCR-ID list; nothing is audited or
   captured that is not on the list.
2. **Capture second** — `03-screenshot-capture.md` produces baseline evidence for every SCR-ID and
   required state.
3. **Extract third** — `04-design-system-extraction.md` documents the current token reality (and
   its divergences) from code.
4. **Audit fourth** — `01-design-audit.md` scores every screen against the heuristics using the
   captured evidence; findings become `docs/factory/audits/DESIGN_AUDIT.md`
   (from `09-templates/design-audit-report.md`).
5. **Handover** — audit + inventory + extracted system (including the Pass 8 Motion & Interaction
   census) feed the design handover (`05-handover/DESIGN_HANDOVER_SCHEMA.md`), which Claude Design
   consumes to produce the redesign. Motion vocabulary and the per-screen Motion lines are
   specified per `03-design/10-motion-design.md`; adaptive behavior per
   `03-design/11-adaptive-layout.md`.
6. **Implement** — `05-material3-modernization.md`, `10-motion-design.md`, `11-adaptive-layout.md`,
   and the remaining docs drive P6 implementation, verified against
   `07-checklists/design-modernization.md`, `07-checklists/accessibility.md`, and the post-build
   design QA pass (`03-design/12-design-qa.md`).

## Design Doctrine (binding on all design work)

1. **Audit before redesign.** No redesign proposal is written until the screen inventory is
   complete, baseline screenshots exist for every SCR-ID, and the current design system has been
   extracted from code. Redesigning from memory or from a single screenshot is forbidden — it
   produces handovers the implementer cannot trace back to evidence.
2. **Tokens before screens.** Fix the design system (color roles, type scale, spacing scale,
   shape scale) before touching individual screens. A screen redesigned against rogue hardcoded
   values will regress the moment the next screen is touched. The divergence report from
   `03-design/04-design-system-extraction.md` is the priority queue.
3. **Material 3 is the default target language.** Every redesign targets Material 3 (M3 themes for
   XML, `androidx.compose.material3` for Compose, dynamic color where brand permits) unless the
   playbook in `06-playbooks/` for that app category explicitly overrides it. Deviations from M3
   must be recorded as decisions in `docs/factory/PROJECT_STATE.md` with rationale.
4. **Accessibility is a gate, not a nice-to-have.** Contrast ≥ 4.5:1 for body text, touch targets
   ≥ 48×48 dp, content descriptions on actionable images, usable at font scale 1.5 — these are
   pass/fail criteria in `07-checklists/accessibility.md`. A redesign that fails them is not done,
   regardless of how it looks.
5. **Evidence or it didn't happen.** Every finding cites a screenshot path; every "after" claim is
   backed by a redesigned-state capture. Visual claims without files under
   `docs/factory/assets/screenshots/` are rejected at handover validation
   (`07-checklists/handover-validation.md`).
6. **Monetization surfaces are first-class screens.** Paywalls, ad placements, and upsell dialogs
   are inventoried, captured, and audited like any other screen — a broken paywall is a Critical
   finding (see severity rules in `03-design/01-design-audit.md`).
7. **Motion is a first-class design concern.** Transitions, list/stagger animation, and
   predictive-back are designed, specified, and verified — not left to defaults. Every animation
   carries a duration token, an easing curve, and a reduced-motion fallback
   (`03-design/10-motion-design.md`); the existing motion is censused at extraction (Pass 8) and
   the specs flow through the handover chain into the design-modernization checklist.
8. **Adaptive layout is a first-class design concern.** Layouts respond to window size classes and
   orientation; tablet/foldable behavior, where in scope, is designed and captured rather than
   left to stretched phone layouts (`03-design/11-adaptive-layout.md`). Inventoried screens are
   audited and captured in both orientations / device classes when adaptive behavior is in scope.

## Inputs these methods assume

- P1–P3 complete: `docs/factory/audits/DISCOVERY.md` and `docs/factory/PROJECT_STATE.md` exist,
  the app builds (`02-engineering/07-build-verification.md`), and a debug APK can be installed.
- A working emulator or device reachable via `adb devices` (setup specifics in
  `03-design/03-screenshot-capture.md`).
- Knowledge baselines: `08-knowledge/design/material3-essentials.md`,
  `08-knowledge/design/mobile-ux-principles.md`, `08-knowledge/design/app-icon-principles.md`.
