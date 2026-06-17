# DESIGN Handover Schema

This schema defines the contract for the engineering→design handover: the document a Claude Code
engineering agent produces (via `01-core/prompts/06-design-handover.md`, lifecycle phase P5) and a
design agent — typically Claude Design running `01-core/prompts/07-design-redesign.md` — consumes.
The consumer has zero access to the codebase, the device, or the producer's session; it must be
able to redesign the app from this document and the attached screenshot set alone. The design
agent's redesign proposal returns to engineering, where prompt 08 converts it into an
IMPLEMENTATION handover (`05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md`).

## File Naming & Versioning

- Instance path: `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md` in the TARGET app repo.
- `<N>` starts at 1; every revision bumps N. Never edit a handover after its status reaches
  Accepted — issue v(N+1) instead. After acceptance only `## Rejection Log` and
  `## Validation Appendix` may be appended (per `01-core/CONVENTIONS.md`).
- Status lifecycle (recorded in the Handover Header): Draft → Delivered → Accepted | Rejected
  (canonical scale in `01-core/CONVENTIONS.md`).
- Each version after v1 must state which version it supersedes and summarize what changed.
- Must pass `07-checklists/handover-validation.md` before the consumer begins work.

## Required Sections

The document must contain exactly these 16 H2 sections, in this order. A missing or empty section is
an automatic validation failure. Two further sections are **whitelisted but not required** and may
be appended after delivery without breaking immutability: `## Rejection Log` (consumer, on reject)
and `## Validation Appendix` (gate re-run evidence). No other section may be added.

### 1. Handover Header

A key/value table with all of: App name, Package ID, App version (versionName/versionCode),
Handover version (vN), Producer (agent/prompt that generated it: `01-core/prompts/06-design-handover.md`),
Date produced (absolute, e.g. 2026-06-10), Factory version (from `VERSION.md`),
Status (Draft / Delivered / Accepted / Rejected), Supersedes (vN-1 or "—").

### 2. App Identity & Brand

Prose plus tables covering: one-sentence value proposition; target audience (who, use context,
device/OS distribution if known); brand constraints that the redesign must respect (logo, mandated
colors, names, legal marks — or an explicit statement "no brand constraints"); tone of voice
(2–4 adjectives with a one-line rationale each). The design agent uses this section to make taste
decisions; vagueness here produces generic redesigns, so every claim must be specific to this app.

### 3. Brand Asset Files

The actual brand source files the redesign must use, as a manifest table — not a prose description
of the brand. When the App Identity & Brand section declares any brand constraint (a logo, mandated
color, wordmark, or legal mark), **at least one logo export is required here**; "no brand
constraints" in §2 is the only way this section may be empty.

| Asset | Path (under `docs/factory/assets/`) | Format | Usage / placement rule |
|---|---|---|---|
| Primary logo | … | vector SVG/XML or PNG @≥xxhdpi | where it may/may not appear |

*Validation:* if §2 declares any brand constraint, ≥1 row of type "logo" is present and its path
exists on disk (rule V11). Every row's path appears in the Asset Manifest.

### 4. Competitive Visual Context

The 2–4 named competitors the redesign positions against, each with at least one annotated
reference screenshot and a one-line "what they do better / what we will do differently" note,
sourced from the design audit's competitive pass. This gives the design agent a visual frame of
reference instead of designing in a vacuum.

| Competitor | Reference screenshot path | Visual strength to match/beat | Our differentiation |
|---|---|---|---|

*Validation:* ≥2 competitor rows, each with a screenshot path that exists on disk and appears in
the Asset Manifest (rule V12).

### 5. Hard Constraints

The non-negotiables a redesign proposal may not violate. Must cover, each in its own subsection
or table:

- **UI toolkit per screen**: for every screen, whether it is XML Views, Jetpack Compose, or hybrid,
  and whether migration is in scope for this cycle. The design agent must know what is cheap vs
  expensive to change.
- **Monetization surfaces that cannot move**: ad placements (format, screen, position), paywall
  entry points, purchase flows — with the reason each is fixed (revenue dependency, network
  policy). Reference `08-knowledge/monetization/ads-patterns.md` conventions for placement names.
- **Platform / legal requirements**: minSdk/targetSdk, edge-to-edge status, required consent or
  disclosure UI, accessibility commitments, store policy constraints from
  `08-knowledge/stores/policy-landmines.md` that touch UI.
- **Performance budgets**: startup time budget, list scroll jank tolerance, APK/AAB size ceiling,
  any screen with known rendering cost limits.

### 6. Current Design System

The extracted as-is tokens, sourced from the `03-design/04-design-system-extraction.md` workflow
output. Four tables, none of which may be empty:

| Table | Required columns |
|---|---|
| Color | Token name, light value (hex), dark value (hex or "no dark theme"), where used |
| Typography | Style name, font family, size (sp), weight, where used |
| Spacing | Token/observed value (dp), where used, consistency notes |
| Shape | Component class, corner radius (dp), where used |

If the app has no formal tokens, document the observed de-facto values ("hardcoded #2196F3 in 14
layouts") — "none defined" is acceptable per cell only with the observed replacement value beside it.

### 7. Motion & Interaction Inventory

The as-is motion census from Extraction Pass 8 (`03-design/04-design-system-extraction.md`), in the
motion vocabulary of `08-knowledge/android/material3-essentials.md` (duration scale 50–600ms
short/medium/long; standard vs emphasized easing; container transform / shared axis / fade-through
patterns; reduced-motion fallback rule). One row per observed transition or gesture-driven
animation, so the redesign proposal can quote a real baseline instead of inventing motion.

| Context (screen/element) | Transition observed | Approx. duration | Easing (if discernible) | Reduced-motion behavior today |
|---|---|---|---|---|

*Validation:* table present with ≥1 row, or an explicit "no app-defined motion observed; system
defaults only" statement (rule V13).

### 8. Adaptive & Device Context

What the redesign must support across form factors and configurations: whether tablet / foldable /
large-screen layouts are in scope **this cycle** (a yes/no the IMPL handover's Adaptive Layout Spec
depends on), the supported orientation(s), the device/OS distribution that justifies the call, and
any window-size-class behavior already present. State edge-to-edge / insets status here if not
already fixed in Hard Constraints.

| Dimension | Value | In scope this cycle? |
|---|---|---|
| Tablet / large screen (≥600dp) | … | Yes / No |
| Foldable | … | Yes / No |
| Orientation | portrait-only / both | — |

*Validation:* the tablet/large-screen row has an explicit Yes/No (rule V14); a "Yes" here makes the
IMPL handover's §Adaptive Layout Spec mandatory downstream.

### 9. Localization & Content Constraints

The locales the redesign must accommodate (longest-string language for layout stress, RTL yes/no)
and the **frozen-vs-redesignable strings declaration** required by `01-core/CONVENTIONS.md` (D13):
which user-facing strings are frozen (legal, contractual, brand, regulated) and may NOT be changed,
versus which are open to copy proposals. Prompt 07 may only propose copy changes against
redesignable strings; prompt 08 emits `COPY-NNN` tickets for accepted ones.

| String / surface | Current text | Frozen or Redesignable | Reason if frozen |
|---|---|---|---|

*Validation:* the frozen-vs-redesignable table is present and every row is classified exactly
Frozen or Redesignable (rule V15); RTL support stated explicitly.

### 10. Screen Inventory

One row per screen, IDs consistent with `docs/factory/audits/` artifacts and the
`09-templates/screen-inventory.md` template:

| Column | Content requirement |
|---|---|
| SCR-ID | Stable ID, e.g. SCR-001; never renumbered across versions |
| Screen name | Human-readable |
| Purpose | One sentence: what the user accomplishes here |
| Traffic priority | High / Medium / Low, with basis (analytics, entry-point analysis, or estimate — say which) |
| Toolkit | XML / Compose / Hybrid |
| States present | Which of default/empty/loading/error/offline this screen has (offline always assessed), per the states contract in `01-core/CONVENTIONS.md` |
| Baseline screenshots | Relative paths under `docs/factory/assets/screenshots/baseline/` (≥1 per screen; light+dark where dark theme exists) |

### 11. Audit Findings

Every design finding from `docs/factory/audits/DESIGN_AUDIT.md` that the redesign should address,
ordered by severity then by traffic priority of the affected screen:

| Column | Content requirement |
|---|---|
| DES-ID | Stable ID matching the design audit, e.g. DES-001 |
| Severity | Critical / High / Medium / Low |
| Screen(s) | SCR-IDs affected, or "Global" |
| Finding | One- to three-sentence description of the problem |
| Evidence | Screenshot path and/or measurable fact (contrast ratio, tap target size in dp) |

Accessibility findings ride their own `A11Y-NNN` stream (per `01-core/CONVENTIONS.md`, the
`A11Y-DES-xxx` format is abolished). Include them in this section with their A11Y-IDs, or in a
clearly labeled A11Y sub-table, so the redesign proposal can address DES and A11Y findings
distinctly. Both streams are subject to the Traceability coverage rule downstream.

### 12. User Insight & Review-Mined Pain Points

What real users complain about, mined from store reviews and any available support/feedback signal
(per `04-aso/workflows/ratings-reviews.md`). This grounds the redesign in lived pain, not only
heuristic findings.

| Theme | Representative quote (verbatim, trimmed) | Screens implicated (SCR-IDs) | Related DES/A11Y finding |
|---|---|---|---|

*Validation:* ≥1 row, or an explicit "no review corpus available (new app / no reviews)" statement
(rule V16). Every SCR-ID and finding ID referenced must resolve.

### 13. Redesign Goals & Non-Goals

Ranked numbered list of goals (most important first), each measurable where possible ("raise all
text contrast to WCAG AA 4.5:1", "Material 3 dynamic color on Android 12+") — and an explicit
Non-Goals list (e.g. "no information-architecture changes", "no new features", "icon handled
separately via `01-core/prompts/10-icon-redesign.md`"). Non-Goals are binding: the consumer must
not propose work listed there.

### 14. Imagery & Illustration Direction

The direction for any non-icon visual content the redesign may introduce or restyle: illustration
style (flat / outline / spot / none), empty-state and onboarding imagery intent, photography
treatment, and any do/don't constraints (no stock-photo people, brand palette only, etc.). This
prevents the proposal from inventing an imagery language engineering cannot source or approve. Icon
and splash are explicitly out of this section — icon is owned by `03-design/06-icon-redesign.md`,
splash by prompt 09's procedure (`03-design/07-splash-redesign.md`).

| Surface | Direction | Constraints / do-not | Source or to-produce |
|---|---|---|---|

*Validation:* present with ≥1 row, or an explicit "no imagery/illustration in scope" statement
(rule V17).

### 15. Per-Screen Briefs

One H3 subsection per screen, covering **every** SCR-ID in the Screen Inventory. Each brief must
contain:

- **Directive**: exactly one of `Keep` (cosmetic token application only), `Fix` (targeted
  corrections, layout intact), `Rethink` (full redesign of the screen within Hard Constraints).
- **Specifics**: what to preserve, what to change, which DES-IDs this screen must resolve, any
  screen-local constraints (e.g. "banner ad anchored bottom, 50dp, cannot move").
- **References**: baseline screenshot paths for this screen.

### 16. Asset Manifest

A single table listing every file referenced anywhere in the handover (screenshots, brand assets,
competitor references, fonts, imagery): path (relative to target repo root), type (screenshot /
logo / icon / font / competitor-ref / other), description, and which section references it. This is
the consumer's packing list — if it is not in the manifest, the consumer must treat it as
nonexistent. (Formerly "Assets Bundle"; renamed Asset Manifest per `01-core/CONVENTIONS.md`.)

## Required Deliverables

1. The handover document at `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md`.
2. The complete baseline screenshot set under `docs/factory/assets/screenshots/baseline/`,
   captured per `03-design/03-screenshot-capture.md` (consistent device profile, light and dark
   where applicable), with every file listed in the Asset Manifest.

## Validation Rules (mechanically checkable)

The handover fails validation if any rule below fails. Run as part of
`07-checklists/handover-validation.md`.

| # | Rule |
|---|---|
| V1 | All 16 required H2 sections present, in order, non-empty (only `## Rejection Log` and `## Validation Appendix` may appear additionally) |
| V2 | Handover Header contains all nine fields; Status is one of the four allowed values |
| V3 | Every Screen Inventory row has all seven columns filled, including `States present` (offline assessed for every screen) |
| V4 | Every screen in the inventory has ≥1 baseline screenshot path AND a Per-Screen Brief subsection |
| V5 | Every Per-Screen Brief has a Directive that is exactly Keep, Fix, or Rethink |
| V6 | Every Critical and High DES finding AND every A11Y finding in Audit Findings is referenced by at least one Per-Screen Brief (or by a Global note in Redesign Goals) |
| V7 | None of the four Current Design System token tables is empty |
| V8 | Every DES-ID / A11Y-ID referenced in briefs exists in the Audit Findings table; every SCR-ID referenced anywhere exists in the Screen Inventory (no unresolved references) |
| V9 | Every path referenced anywhere in the document appears in the Asset Manifest AND exists on disk in the target repo (no dangling asset references) |
| V10 | No placeholder tokens (`{{...}}`), no "TBD"/"TODO" strings anywhere in the document |
| V11 | If App Identity & Brand declares any brand constraint, Brand Asset Files has ≥1 logo-type row whose path exists on disk; "no brand constraints" is the only empty-allowed case |
| V12 | Competitive Visual Context has ≥2 competitor rows, each with an on-disk screenshot path |
| V13 | Motion & Interaction Inventory has ≥1 row or the explicit "system defaults only" statement |
| V14 | Adaptive & Device Context states an explicit Yes/No for tablet / large-screen scope |
| V15 | Localization & Content Constraints classifies every listed string exactly Frozen or Redesignable; RTL support stated |
| V16 | User Insight & Review-Mined Pain Points has ≥1 row or the explicit "no review corpus" statement; referenced IDs resolve |
| V17 | Imagery & Illustration Direction has ≥1 row or the explicit "no imagery in scope" statement |

## Acceptance Criteria (consumer's bar)

The consumer evaluates from its own seat — no codebase access, no producer to ask. Accept only if
all of the following hold; otherwise reject:

1. **The single-question test**: "Could I redesign this app from this document alone, without
   asking a single question?" This bar is now *truly* deliverable: the brand source files
   (§3), competitive frame (§4), motion baseline (§7), adaptive scope (§8), frozen-string
   declaration (§9), review-mined pain (§12), and imagery direction (§14) close the gaps that
   previously forced the design agent to ask. Any fact the consumer would still need to ask for is
   grounds for rejection.
2. Hard Constraints are concrete enough to design against (exact ad sizes/positions, exact toolkit
   per screen) — not generalities like "keep monetization intact".
3. Goals and Non-Goals are unambiguous and non-overlapping; for every screen it is clear what
   success looks like, and the frozen-vs-redesignable string boundary is unambiguous.
4. Baseline screenshots are legible, current (match the stated app version), and sufficient to
   understand every screen's existing state, including its offline state where present.
5. All Validation Rules V1–V17 pass on the consumer's re-run of the gate.

## Rejection Protocol

1. The consumer appends a `## Rejection Log` section to the handover file (one of only two
   permitted post-delivery appends, the other being `## Validation Appendix`) with one row per
   reason:

   | Date | Rejected by | Rule / criterion violated | Detail |
   |---|---|---|---|

2. The consumer sets Status to Rejected in the Handover Header and stops work.
3. The producer issues `DESIGN_HANDOVER_v<N+1>.md` addressing every logged reason, with
   Supersedes set to vN. The rejected file is never deleted or edited further.
4. Both events are recorded in `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md`.

## Producer & Consumer Workflow

1. **Producer** (prompt 06) drafts the document with Status: Draft, assembling inputs from
   `docs/factory/audits/DESIGN_AUDIT.md`, the screen inventory, the design-system extraction, and
   the baseline screenshot capture.
2. Producer self-runs `07-checklists/handover-validation.md` (rules V1–V17). Fix everything; do
   not deliver a known-failing handover hoping the consumer won't notice.
3. Producer sets Status: Delivered, records delivery in `docs/factory/PROJECT_STATE.md`
   (Artifacts table) and `docs/factory/CHANGELOG.md`.
4. **Consumer** (prompt 07 / Claude Design) re-runs the validation gate, then evaluates the
   Acceptance Criteria from its own seat.
5. On accept: consumer sets Status: Accepted and begins the redesign, citing
   `DESIGN_HANDOVER_v<N>` in its proposal. On reject: follow the Rejection Protocol below.
6. The redesign proposal the consumer returns is the input to
   `01-core/prompts/08-implementation-handover.md`, which closes the loop with an
   IMPLEMENTATION handover.

## Common Failure Modes

| Failure | Symptom downstream | Prevention |
|---|---|---|
| Constraints stated as vibes ("keep ads working") | Design proposal moves a revenue-critical placement; whole proposal rejected | Hard Constraints must name format, screen, position, and reason per surface |
| Screenshots from an older build | Consumer designs against UI that no longer exists | V9 existence check + Acceptance Criterion 4 (screenshots match stated app version) |
| Briefs only for "interesting" screens | Consumer invents direction for uncovered screens | V4: every SCR-ID gets a brief, even if Directive: Keep |
| Audit findings copied without evidence | Consumer cannot judge severity or verify the fix | Evidence column mandatory (path or measurable fact) in Section 6 |
| Goals and Non-Goals overlap or contradict | Consumer ships work that engineering refuses to implement | Acceptance Criterion 3; producer reviews goals against constraints before delivery |

## Example Skeleton

```markdown
# DESIGN Handover — NotesPro v3.2.0

## 1. Handover Header
| Field | Value |
|---|---|
| App | NotesPro (com.example.notespro) |
| App version | 3.2.0 (320) |
| Handover version | v1 |
| Producer | 01-core/prompts/06-design-handover.md |
| Date produced | 2026-06-10 |
| Factory version | 1.1.0 |
| Status | Delivered |
| Supersedes | — |

## 2. App Identity & Brand
Value prop: fastest offline note capture for field technicians. Brand constraints: wordmark + #1B5E20 brand green are mandatory. Tone: precise, rugged, fast.

## 3. Brand Asset Files
| Asset | Path | Format | Usage / placement rule |
|---|---|---|---|
| Primary logo | docs/factory/assets/brand/logo-notespro.svg | vector SVG | top-start only; never recolored |

## 4. Competitive Visual Context
| Competitor | Reference screenshot path | Visual strength to match/beat | Our differentiation |
|---|---|---|---|
| FieldNotes | docs/factory/assets/competitors/fieldnotes-list.png | clean M3 list density | faster capture, offline-first badge |

## 5. Hard Constraints
| Constraint | Detail | Reason |
|---|---|---|
| Banner ad on SCR-001 | AdMob adaptive banner, bottom-anchored | 62% of ad revenue |

## 6. Current Design System
| Color token | Light | Dark | Used in |
|---|---|---|---|
| (none defined; observed) #2196F3 | #2196F3 | no dark theme | toolbar, FAB, 14 layouts |

## 7. Motion & Interaction Inventory
| Context | Transition observed | Approx. duration | Easing | Reduced-motion behavior today |
|---|---|---|---|---|
| Note open (SCR-001→SCR-002) | cross-fade activity transition | ~300ms | system default | none (ignores Remove animations setting) |

## 8. Adaptive & Device Context
| Dimension | Value | In scope this cycle? |
|---|---|---|
| Tablet / large screen (≥600dp) | single-pane stretched | No |
| Foldable | untested | No |
| Orientation | portrait-only | — |

## 9. Localization & Content Constraints
Locales: en-US, de-DE. RTL: no. Frozen-vs-redesignable strings:
| String / surface | Current text | Frozen or Redesignable | Reason if frozen |
|---|---|---|---|
| Consent dialog body | "We store notes locally…" | Frozen | legal/privacy review |
| SCR-001 empty state | "No notes yet" | Redesignable | — |

## 10. Screen Inventory
| SCR-ID | Name | Purpose | Traffic | Toolkit | States present | Baseline |
|---|---|---|---|---|---|---|
| SCR-001 | Note list | Browse and open notes | High (analytics) | XML | default, empty, offline | docs/factory/assets/screenshots/baseline/scr-001-light.png |

## 11. Audit Findings
| DES-ID | Severity | Screen(s) | Finding | Evidence |
|---|---|---|---|---|
| DES-003 | Critical | SCR-001 | Body text contrast 2.9:1 on list items | scr-001-light.png; measured #9E9E9E on #FFFFFF |
| A11Y-001 | High | SCR-001 | FAB tap target 40dp (< 48dp) | measured in layout inspector |

## 12. User Insight & Review-Mined Pain Points
| Theme | Representative quote | Screens | Related finding |
|---|---|---|---|
| Hard to read outdoors | "can't see the gray text in sunlight" | SCR-001 | DES-003 |

## 13. Redesign Goals & Non-Goals
1. All text meets WCAG AA (resolves DES-003); tap targets ≥48dp (resolves A11Y-001). ...
Non-Goals: no IA changes; icon out of scope (prompt 10); splash out of scope (prompt 09).

## 14. Imagery & Illustration Direction
| Surface | Direction | Constraints / do-not | Source or to-produce |
|---|---|---|---|
| Empty states | flat outline spot illustration, brand green | no stock photos | to-produce |

## 15. Per-Screen Briefs
### SCR-001 — Note list
Directive: Fix. Resolve DES-003 and A11Y-001; preserve bottom banner position; ...

## 16. Asset Manifest
| Path | Type | Description | Referenced in |
|---|---|---|---|
| docs/factory/assets/brand/logo-notespro.svg | logo | NotesPro wordmark | §3, §15 |
| docs/factory/assets/competitors/fieldnotes-list.png | competitor-ref | FieldNotes list | §4 |
| docs/factory/assets/screenshots/baseline/scr-001-light.png | screenshot | Note list, light | §10, §11, §12, §15 |
```
