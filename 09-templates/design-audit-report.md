# Template — Design Audit Report

This template produces the UX/UI health assessment of a target Android app: the **per-screen
H1–H9 scorecard** (the canonical rubric from `03-design/01-design-audit.md`), per-screen and
systemic findings, the extracted current design system, accessibility results in their own A11Y
stream, and the raw User-Insight / Competitive-Context material the redesign reasons from. The
instance feeds `01-core/prompts/06-design-handover.md` directly — a Claude Design session will
redesign from this document plus `SCREEN_INVENTORY.md`, so findings must be screenshot-evidenced
and the redesign recommendation must be unambiguous.

## Usage

- **Instantiated by:** the design auditor agent running `01-core/prompts/05-design-audit.md`
  (drawing on the methods in `03-design/01-design-audit.md` and
  `03-design/04-design-system-extraction.md`).
- **Phase:** P5 — Design Audit & Handover.
- **Instance path:** `docs/factory/audits/DESIGN_AUDIT.md` in the TARGET app repo.
- **Prerequisite:** `docs/factory/audits/SCREEN_INVENTORY.md` must already exist with baseline
  screenshots captured; this report references screens by their SCR-xxx IDs.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / applicationId | Solar Camera Pro / com.solarcamera.pro |
| `{{AUDIT_DATE}}` | Date audit completed, ISO 8601 | 2026-06-10 |
| `{{AUDITOR_ROLE}}` | Persona from prompt 05 Role section | Senior Product Designer (Android/M3) |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{COMMIT_SHA}}` | Git SHA the screenshots were captured from | 9f3c1a7e… |
| `{{UX_VERDICT}}` | One-line UX health verdict: Modern / Dated / Severely Dated / Broken + qualifier | Severely Dated — Holo-era patterns throughout |
| `{{REDESIGN_RECOMMENDED}}` | Y or N | Y |
| `{{REDESIGN_SCOPE}}` | If Y: Full / Targeted (list screens) / Theming-only | Targeted — SCR-001, SCR-003, SCR-007 |
| `{{SCREEN_COUNT}}` | Total screens in `SCREEN_INVENTORY.md` | 14 |
| `{{SCORED_SCREEN_COUNT}}` | Screens actually scored on H1–H9 (≤ `{{SCREEN_COUNT}}`; rest in Coverage & Limitations) | 12 |
| `{{SCORECARD_ROWS}}` | Per-screen H1–H9 scorecard rows (one per scored SCR-id; format in guidance) | (table rows) |
| `{{COLOR_TOKEN_TABLE_ROWS}}` / `{{TYPE_TOKEN_TABLE_ROWS}}` / `{{SPACING_TOKEN_TABLE_ROWS}}` | Extracted design-system table rows (one per token found in code) | (table rows) |
| `{{SYSTEMIC_FINDING_1_TITLE}}` / `{{SYSTEMIC_FINDING_1_BODY}}` | Title + prose for each systemic (SYS) finding; repeat the H3 block per systemic finding | No app-wide dark theme / (prose) |
| `{{MOBILE_UX_PRINCIPLE_ROWS}}` | Pass/Partial/Fail rows for the mobile-UX principles summary | (table rows) |
| `{{A11Y_FINDING_COUNT}}` | Count of accessibility findings (A11Y stream) | 6 |
| `{{COVERAGE_GAP_ROWS}}` | Rows for screens/states not fully scored, with reasons | (table rows) |
| `{{USER_INSIGHT_BODY}}` | Review-mined pain points / user vocabulary feeding prompt 06 | (prose + bullets) |
| `{{COMPETITIVE_CONTEXT_BODY}}` | Competitor visual/UX patterns worth matching or beating, feeding prompt 06 | (prose + bullets) |
| `{{HANDOVER_PRIORITIES}}` | Ordered list of what the redesign must fix first | 1. Navigation model, 2. Color contrast… |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Design Audit — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Audit date | {{AUDIT_DATE}} |
| Auditor role | {{AUDITOR_ROLE}} |
| Factory version | {{FACTORY_VERSION}} |
| Commit audited | `{{COMMIT_SHA}}` |

## Executive Summary

**UX health verdict:** {{UX_VERDICT}}

<!-- 3–6 sentences. Lead with how the app feels relative to 2026 Material 3 expectations
     (08-knowledge/design/material3-essentials.md is the bar). Name the 2–3 issues that most
     hurt users or conversion. Quantify where possible ("4 of 14 screens have no empty state"). -->

**Redesign recommended:** {{REDESIGN_RECOMMENDED}} — **Scope:** {{REDESIGN_SCOPE}}

<!-- The recommendation must be decidable from this line alone. If N, say what targeted fixes
     replace a redesign. If Y, the scope line tells prompt 06 exactly what to hand over. -->

## Screen Inventory Reference

Full inventory: `docs/factory/audits/SCREEN_INVENTORY.md` — **{{SCREEN_COUNT}} screens** audited.

<!-- One short paragraph: which screens carry the most traffic/monetization weight and therefore
     got the deepest review. Do not duplicate the inventory table here. -->

## Per-Screen Heuristic Scorecard (H1–H9)

<!-- THE rubric is the nine heuristics defined in 03-design/01-design-audit.md (H1 Visual
     hierarchy · H2 Consistency · H3 Feedback · H4 Navigation clarity · H5 Error handling ·
     H6 Empty states · H7 Loading states · H8 Touch ergonomics · H9 Platform conventions). Score
     EVERY inventoried screen 1 (broken) to 5 (exemplary) per heuristic, using the per-screen
     procedure and scoring rubric in that method file — do not restate the rubric here, cite it.
     One row per scored SCR-id. Mark `n/a` for a heuristic that cannot apply to a screen (excluded
     from Avg, never scored 5 by default). Priority = traffic priority (H/M/L) copied from
     SCREEN_INVENTORY.md — it weights which low scores drive redesign scope. Any cell scored 1 or 2
     MUST map to at least one DES finding below. The old per-app 10-heuristic table is abolished. -->

**{{SCORED_SCREEN_COUNT}} of {{SCREEN_COUNT}} screens scored** (unscored screens are listed in
Coverage & Limitations with reasons).

| SCR-ID | H1 | H2 | H3 | H4 | H5 | H6 | H7 | H8 | H9 | Avg | Priority |
|---|---|---|---|---|---|---|---|---|---|---|---|
| SCR-001 | 3 | 2 | 4 | 4 | 2 | n/a | 3 | 4 | 2 | 3.0 | H | <!-- (example — replace) -->
{{SCORECARD_ROWS}}

## Coverage & Limitations

<!-- The honesty section. Record: (a) every inventoried screen NOT scored above and why
     (e.g. unreachable behind a paywall purchase, server-gated state); (b) every required state
     that could not be captured (carry over from SCREEN_INVENTORY.md Capture Gaps) and how it was
     handled (simulated / accepted); (c) any heuristic deliberately not assessed on a screen.
     Silence is treated as a gap — if it is not here, the reader assumes full coverage. Each
     accepted gap that carries risk must also appear in docs/factory/reports/RISK_REGISTER.md. -->

| Screen / state | What was not covered | Reason | Handling |
|---|---|---|---|
| SCR-011 (paywall) | error + offline states | Purchase flow needs a sandbox account not available | Simulated error via charles proxy; offline accepted, RISK-006 | <!-- (example — replace) -->
{{COVERAGE_GAP_ROWS}}

## Mobile-UX Principles Summary

<!-- A SEPARATE Pass/Partial/Fail summary of the app-wide mobile-UX principles from
     08-knowledge/design/mobile-ux-principles.md — NOT a second scorecard, no 1–5 scores. One row
     per principle; Verdict ∈ Pass / Partial / Fail; Evidence names a SCR-id or systemic finding.
     Use this to capture app-level posture (thumb-zone reachability, one-handed use, gesture
     affordances, motion restraint) that the per-screen scorecard does not surface. -->

| Principle | Verdict | Evidence |
|---|---|---|
| Primary actions reachable in the thumb zone | Partial | H-screens OK; SCR-007 CTA top-right — DES-012 | <!-- (example — replace) -->
{{MOBILE_UX_PRINCIPLE_ROWS}}

## Findings Register

<!-- One row per finding, ID DES-001 ascending (zero-padded 3 per CONVENTIONS.md D2), ordered by
     severity. Class = SCR (per-screen) or SYS (systemic — also gets a paragraph below and lists
     all affected SCR-ids). Heuristic = the H1–H9 it violates (per 03-design/01-design-audit.md).
     Evidence = repo-relative screenshot under docs/factory/assets/screenshots/baseline/ following
     SCR-<id>-<state>-<theme>.png — every finding needs a visible artifact. Severity =
     Critical/High/Medium/Low; Status starts "Not Started". Accessibility findings do NOT go here —
     they live in the A11Y stream below. -->

| ID | Class | Screens | Severity | Heuristic | Evidence | Recommendation | Status |
|---|---|---|---|---|---|---|---|
| DES-014 | SCR | SCR-003 | High | H5 — Error handling | `docs/factory/assets/screenshots/baseline/SCR-003-error-light.png` | Offline error state with retry; replace infinite spinner | Not Started | <!-- (example — replace) -->

## Systemic Findings

<!-- Issues that no single screen owns: navigation model, inconsistent back behavior, mixed
     XML/Compose styling drift, missing dark theme, no edge-to-edge. One H3 + short paragraph
     per systemic finding, each referencing its DES-xxx register row. These usually drive the
     redesign-scope decision — argue the connection explicitly. -->

### {{SYSTEMIC_FINDING_1_TITLE}}

{{SYSTEMIC_FINDING_1_BODY}}

## Current Design System Summary

<!-- Extracted from code per 03-design/04-design-system-extraction.md. Purpose: show the
     redesigner what exists today so the Design Handover can map old → new tokens. Capture what
     IS, not what should be. If a category is absent (e.g. no type scale, ad-hoc sizes), record
     that as a row — absence is a finding. -->

### Color

| Token / usage | Value | Defined at | Notes |
|---|---|---|---|
| `colorPrimary` | `#3F51B5` | `res/values/colors.xml:4` | Holo Indigo; fails 4.5:1 on white in 3 usages | <!-- (example — replace) -->
{{COLOR_TOKEN_TABLE_ROWS}}

### Typography

| Style | Font / size / weight | Defined at | Notes |
|---|---|---|---|
| `TextAppearance.Body` | Roboto 14sp Regular | `res/values/styles.xml:21` | Used inconsistently; 9 hardcoded sp values elsewhere | <!-- (example — replace) -->
{{TYPE_TOKEN_TABLE_ROWS}}

### Spacing & Shape

| Token / pattern | Value | Notes |
|---|---|---|
| Margin grid | none — ad-hoc 4/6/10/12/16dp mix | No spacing scale exists | <!-- (example — replace) -->
{{SPACING_TOKEN_TABLE_ROWS}}

## Accessibility Findings (A11Y stream)

**{{A11Y_FINDING_COUNT}} findings.** Method: `03-design/09-accessibility-review.md` +
`07-checklists/accessibility.md`.

<!-- Accessibility findings have their OWN ID stream — A11Y-001 ascending (format per
     CONVENTIONS.md D2; the old A11Y-DES-xxx format is abolished). They are NOT DES findings and
     do not appear in the register above. Same evidence rules as DES-xxx. When a problem is both a
     design defect and an accessibility-gate failure, file a DES finding for the design fix AND a
     separate A11Y finding for the gate, cross-referenced by ID. Check at minimum: contrast ratios,
     touch target sizes (48dp), content descriptions, TalkBack traversal order, text scaling at
     200%. -->

| ID | Screen | Severity | Issue | Evidence | Recommendation | Status |
|---|---|---|---|---|---|---|
| A11Y-001 | SCR-001 | High | FAB has no `contentDescription`; TalkBack reads "unlabeled button" | `docs/factory/assets/screenshots/baseline/scr-001-default.png` | Add description; verify full TalkBack pass | Not Started | <!-- (example — replace) -->

## Handover Inputs — Raw Material for Prompt 06

<!-- This is the RAW EVIDENCE the Design Handover (06) turns into its "User Insight & Review-Mined
     Pain Points" and "Competitive Visual Context" sections (per CONVENTIONS.md D15). It is source
     material, not directives — the directive handoff is the next section. Keep it concrete and
     attributed: real review excerpts, real competitor screens. -->

### User Insight & Review-Mined Pain Points

{{USER_INSIGHT_BODY}}

<!-- Pain points mined from the app's own store reviews and support channels: what users struggle
     with, the words they use (feeds copy/IA), recurring complaints tied to specific screens (cite
     SCR-ids). Quote 3–6 representative review excerpts with their rating. This is the same raw
     material the ASO own-app review-mining table draws on — keep them consistent. -->

### Competitive Visual Context

{{COMPETITIVE_CONTEXT_BODY}}

<!-- 3–5 direct competitors: the visual/UX patterns worth matching or beating (navigation model,
     onboarding, empty-state treatment, motion). Reference screenshots if captured under
     docs/factory/assets/screenshots/competitors/. State what the redesign should adopt vs
     differentiate from — the redesigner reasons from this in prompt 07. -->

## Inputs for Design Handover (Prompt 06 handoff)

<!-- What 01-core/prompts/06-design-handover.md reads first. Be directive: ordered priorities,
     hard constraints (brand colors that must stay, screens out of scope), and which findings
     are table-stakes vs aspirational. -->

- **Redesign priorities (ordered):** {{HANDOVER_PRIORITIES}}
- **Hard constraints:** <!-- brand/legal/owner constraints the redesign may not violate -->
- **Out of scope this cycle:** <!-- screens or findings explicitly excluded, with reasons; mirror
  any accepted risk into docs/factory/reports/RISK_REGISTER.md -->
- **Must-pass gates:** all Critical/High DES-xxx and A11Y-xxx findings addressed in the handover's
  screen specs, validated by `07-checklists/design-modernization.md`.
