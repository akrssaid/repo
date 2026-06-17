# IMPLEMENTATION Handover Schema

This schema defines the contract for the design→engineering handover: the document produced by
`01-core/prompts/08-implementation-handover.md` (lifecycle phase P5, closing the design round
trip) and consumed by `01-core/prompts/09-ui-implementation.md` in phase P6. The producer
translates an accepted redesign proposal into executable engineering work; the consumer is an
engineering agent in a fresh session that has the codebase but none of the design conversation.
Every design decision must therefore arrive as an exact value, a complete spec, or a pointer to a
file that exists — never as "as discussed" or "per the design".

## File Naming & Versioning

- Instance path: `docs/factory/handovers/IMPLEMENTATION_HANDOVER_v<N>.md` in the TARGET app repo.
- `<N>` starts at 1; every revision bumps N. Never edit a handover after its status reaches
  Accepted — issue v(N+1) instead. After acceptance only `## Rejection Log` and
  `## Validation Appendix` may be appended (per `01-core/CONVENTIONS.md`).
- Status lifecycle (recorded in the Handover Header): Draft → Delivered → Accepted | Rejected
  (canonical scale in `01-core/CONVENTIONS.md`).
- The header must name the accepted DESIGN handover version AND the accepted REDESIGN_PROPOSAL
  version this document implements (e.g. "Implements: DESIGN_HANDOVER_v2 + REDESIGN_PROPOSAL_v2").
- Must pass `07-checklists/handover-validation.md` before the consumer begins work.

## Required Sections

The document must contain exactly these 10 H2 sections, in this order. A missing or empty section
is an automatic validation failure. Two further sections are **whitelisted but not required** and
may be appended after delivery without breaking immutability: `## Rejection Log` (consumer, on
reject) and `## Validation Appendix` (gate re-run evidence). No other section may be added.

### 1. Handover Header

A key/value table with all of: App name, Package ID, App version (versionName/versionCode),
Handover version (vN), Producer (`01-core/prompts/08-implementation-handover.md`), Date produced
(absolute), Factory version, Status (Draft / Delivered / Accepted / Rejected), Supersedes
(vN-1 or "—"), Implements (the accepted DESIGN handover version AND the accepted REDESIGN_PROPOSAL
version this document converts).

### 2. Design System Specification

The complete target token set with exact values — the consumer implements the theme from this
section first (theme-first rule, see Implementation Sequence). Required tables:

| Table | Required columns |
|---|---|
| Color roles | Role (Material 3 role name, e.g. primary, onSurfaceVariant), light value (hex), dark value (hex), notes (dynamic-color behavior on Android 12+) |
| Type scale | Style (M3 scale name, e.g. titleLarge), font family, size (sp), line height (sp), weight |
| Shape | Scale step (extraSmall…extraLarge), corner radius (dp), example components |
| Spacing | Token name, value (dp), usage rule |

Plus a **code-mapping table** — token → implementation target — so values land in the right place:

| Token | Compose target | XML target (if hybrid) |
|---|---|---|
| e.g. primary | `ColorScheme.primary` in `ui/theme/Color.kt` + `Theme.kt` | `colorPrimary` in `res/values/themes.xml` |

Both light **and** dark values are mandatory for every color role. If the app drops dark theme,
state that as an explicit decision in this section, not by omission.

### 3. Component Specifications

One H3 subsection per shared component (anything used on ≥2 screens or extracted per
`03-design/08-component-library.md`). Each must define:

- **Anatomy**: the parts of the component and which tokens each part consumes.
- **States**: enabled, disabled, pressed, focused, selected, error, loading — every state that
  applies, with the visual delta per state.
- **Slots / params**: the component's API surface (Compose parameters or XML attributes),
  including defaults.

### 4. Motion Specification

The complete motion contract converted from the REDESIGN_PROPOSAL's per-screen Motion lines,
expressed in the vocabulary of `08-knowledge/android/material3-essentials.md` (duration scale
50–600ms; standard vs emphasized easing; container transform / shared axis / fade-through). One row
per distinct transition/animation the implementation must produce:

| Context (screen/element) | Transition | Duration | Easing | Reduced-motion fallback |
|---|---|---|---|---|

Every row's reduced-motion fallback is mandatory (e.g. "instant state change; no translate when
Settings → Remove animations is on"). Each row must be picked up by a ticket via that ticket's
`Motion:` field, or be explicitly listed in Out of Scope as a rejected proposal Motion line — this
is the D14 V-rule (every proposal Motion line maps to a ticket or is explicitly rejected, V11).

### 5. Adaptive Layout Spec

**Required only when the DESIGN handover's Adaptive & Device Context marks tablet / large screen
in scope.** When tablet is out of scope, this section contains the single line "Tablet/large-screen
out of scope per DESIGN_HANDOVER_v<N> §Adaptive & Device Context" and nothing more. When in scope,
specify per affected screen: the window-size-class breakpoint behavior (compact / medium / expanded),
the pane strategy (single-pane vs list-detail), navigation surface change (bottom bar → rail/drawer),
and exact dp breakpoints. Acceptance criteria for the corresponding tickets must be **runnable on a
tablet AVD** (state the AVD profile).

| Screen | Compact (<600dp) | Medium (600–840dp) | Expanded (≥840dp) | Nav surface |
|---|---|---|---|---|

### 6. Ticket Breakdown

One H3 subsection per ticket. Each ticket must contain every field below:

| Field | Content requirement |
|---|---|
| ID | IMPL-xxx (UI work) or COPY-xxx (string changes) — see ticket types below; stable across versions |
| Title | Imperative, specific ("Restyle note list rows to M3 ListItem") |
| Screen | SCR-ref from the DESIGN handover's Screen Inventory, or "Global" |
| DES findings addressed | DES-xxx / A11Y-xxx list, or "—" for pure-infrastructure tickets |
| Files affected (expected) | Repo-relative paths the producer expects to change; "new file" entries marked as such |
| Spec | Full detail inline, or a pointer to the exact section/file containing it (pointer target must exist) |
| Motion | The Motion Specification row(s) this ticket realizes, or "—" if none |
| A11y criteria | 1–2 screen-specific accessibility checks (e.g. "FAB ≥48dp touch target; talkback reads label 'Add note'") |
| Acceptance criteria | Verifiable checklist — observable on device or in code, not "looks good" |
| Effort | S / M / L / XL per the canonical scale |
| Dependencies | IMPL-xxx / COPY-xxx IDs that must complete first, or "none" |

**Ticket types** (per `01-core/CONVENTIONS.md` ID registry):
- **IMPL-NNN** — UI/engineering implementation work.
- **COPY-NNN** — string changes only (D13). Emitted for each copy change the REDESIGN_PROPOSAL
  proposed against a *redesignable* string (never a frozen one, per the DESIGN handover's
  Localization & Content Constraints). A COPY ticket's Spec gives the exact before→after text and
  the string resource key; its A11y criteria cover any label/contentDescription impact.

### 7. Implementation Sequence

A single ordered table (position, ticket ID, rationale) covering every ticket exactly once —
IMPL and COPY tickets alike. **Theme-first rule**: tickets that implement Section 2 (theme/tokens)
must occupy the earliest positions, before any screen-level ticket — screens are restyled against
the new theme, never against ad-hoc values. The sequence must be consistent with every ticket's
Dependencies field (no ticket ordered before its dependency).

### 8. Asset Deliverables

One row per asset the implementation needs (icons, illustrations, lottie files, fonts):

| Field | Content requirement |
|---|---|
| ID | ASSET-xxx |
| Type | icon / illustration / animation / font / image |
| Spec | What it depicts, style constraints, token colors used |
| Format | File format + size/density variants required (e.g. vector XML; or PNG @ mdpi–xxxhdpi) |
| Destination path | Exact repo path, e.g. `app/src/main/res/drawable/ic_empty_notes.xml` |
| Status | Delivered (file exists at path) / To produce (which ticket produces it) |

### 9. Out of Scope

Explicit list of what this handover does **not** cover (deferred findings, screens kept as-is,
features, **and any REDESIGN_PROPOSAL Motion line or copy change deliberately not ticketed**). Each
deferred Critical/High DES finding, every A11Y finding, and every rejected Motion line must appear
here with a one-line rationale and where it is tracked (e.g. `docs/factory/PROJECT_STATE.md`
backlog). Silence is not deferral.

### 10. Traceability Matrix

A single matrix covering DESIGN findings, accessibility findings, and copy changes — one row per
DES-ID and A11Y-ID from the DESIGN handover's Audit Findings, plus one row per copy change proposed
by the REDESIGN_PROPOSAL:

| Finding / change ID | Type (DES / A11Y / COPY) | Severity | Resolved by (IMPL-xxx / COPY-xxx list) | Or status |
|---|---|---|---|---|

Every DES and A11Y finding maps to ≥1 ticket OR carries status `Deferred` with a rationale
cross-referenced in Out of Scope. Every proposed copy change maps to a COPY ticket OR is marked
`Rejected (frozen string)` / `Deferred`. No finding or proposed copy change may be absent from this
table.

## Required Deliverables

1. The handover document at `docs/factory/handovers/IMPLEMENTATION_HANDOVER_v<N>.md`.
2. All assets marked Delivered in Asset Deliverables, present at their destination paths.
3. Any external spec files pointed to by ticket Spec fields (e.g. redesign proposal exports under
   `docs/factory/assets/`), present at the referenced paths.

## Validation Rules (mechanically checkable)

| # | Rule |
|---|---|
| V1 | All 10 required H2 sections present, in order, non-empty (only `## Rejection Log` and `## Validation Appendix` may appear additionally) |
| V2 | Handover Header contains all ten fields, including a resolvable Implements reference naming both the DESIGN handover and the REDESIGN_PROPOSAL versions |
| V3 | Every color role in Design System Specification has both a light and a dark value (or the section contains an explicit dark-theme-dropped decision) |
| V4 | Every token row appears in the code-mapping table |
| V5 | Every ticket has all eleven fields including `Motion:` and `A11y criteria:`; Acceptance criteria non-empty; Effort ∈ {S,M,L,XL}; every ID is IMPL-xxx or COPY-xxx |
| V6 | Every ticket appears exactly once in the Implementation Sequence; no ticket precedes one of its Dependencies; theme tickets precede all screen tickets |
| V7 | No ticket references a SCR-ID absent from the DESIGN handover's Screen Inventory; no unresolved DES-xxx, A11Y-xxx, IMPL-xxx, COPY-xxx, or ASSET-xxx references anywhere |
| V8 | Traceability Matrix contains every DES and A11Y finding and every proposed copy change; each Critical/High DES finding and each A11Y finding is mapped to ≥1 ticket or marked Deferred with a rationale that also appears in Out of Scope; each copy change maps to a COPY ticket or is marked Rejected/Deferred |
| V9 | Every Spec pointer and every Delivered asset path exists on disk; every destination path is syntactically valid for the project layout |
| V10 | No placeholder tokens (`{{...}}`), no "TBD"/"TODO" strings anywhere in the document |
| V11 | Every Motion Specification row is realized by a ticket's `Motion:` field OR is explicitly listed in Out of Scope as a rejected proposal Motion line (D14); every Motion row has a non-empty reduced-motion fallback |
| V12 | When the DESIGN handover marks tablet/large screen in scope, Adaptive Layout Spec is populated and the affected tickets' Acceptance criteria name a tablet AVD; when out of scope, the section is the single out-of-scope line |

## Acceptance Criteria (consumer's bar)

Accept only if all hold; otherwise reject:

1. **The five-minute test**: an engineer (or engineering agent) could start ticket 1 within 5
   minutes of reading — first file to open and first change to make are unambiguous.
2. Every ticket is self-sufficient: spec + acceptance criteria decide all visual and behavioral
   questions without consulting the producer.
3. The theme can be implemented end-to-end from Section 2 alone — no missing roles, no "pick
   something close".
4. The sequence is executable as written: following it never blocks on an unfinished dependency
   or an undelivered asset.
5. Every proposal Motion line is either ticketed (with duration, easing, and reduced-motion
   fallback) or explicitly rejected; every COPY ticket targets a redesignable (not frozen) string.
6. All Validation Rules V1–V12 pass on the consumer's re-run of
   `07-checklists/handover-validation.md`.

## Rejection Protocol

1. The consumer appends a `## Rejection Log` section to the handover file (one of only two
   permitted post-delivery appends, the other being `## Validation Appendix`), one row per reason:

   | Date | Rejected by | Rule / criterion violated | Detail |
   |---|---|---|---|

2. The consumer sets Status to Rejected and stops work.
3. The producer issues `IMPLEMENTATION_HANDOVER_v<N+1>.md` addressing every logged reason, with
   Supersedes set to vN. The rejected file is never deleted or edited further.
4. Both events are recorded in `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md`.

## Producer & Consumer Workflow

1. **Producer** (prompt 08) drafts with Status: Draft, working from the accepted DESIGN handover
   and the design agent's redesign proposal. Every proposal decision is converted into a token
   value, a component spec, or a ticket — nothing remains only in prose.
2. Producer self-runs `07-checklists/handover-validation.md` (rules V1–V12), paying particular
   attention to V8 (traceability over DES + A11Y + COPY), V9 (every pointer resolves), and V11
   (every proposal Motion line ticketed or rejected).
3. Producer sets Status: Delivered, records delivery in `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md`.
4. **Consumer** (prompt 09) re-runs the gate, then dry-runs the five-minute test on ticket 1
   before accepting.
5. On accept: consumer sets Status: Accepted, executes tickets in Implementation Sequence order,
   and updates each ticket's status in `docs/factory/PROJECT_STATE.md` as work proceeds (the
   handover itself stays immutable). On reject: follow the Rejection Protocol below.
6. Scope discovered mid-implementation (missing spec, new finding) is **not** patched into the
   accepted handover — it is logged in PROJECT_STATE and either absorbed under an existing
   ticket's acceptance criteria or deferred to v(N+1)/next cycle.

## Common Failure Modes

| Failure | Symptom downstream | Prevention |
|---|---|---|
| Spec by reference to the design conversation ("as discussed") | Consumer has no conversation; ticket stalls | V9: every Spec pointer must resolve to a file/section that exists |
| Screen tickets ordered before theme tickets | Screens styled with ad-hoc values, restyled twice | V6 theme-first rule in Implementation Sequence |
| Dark values missing ("derive from light") | Consumer invents a dark theme the designer never approved | V3: both values mandatory per role |
| Acceptance criteria of the form "matches design" | Unverifiable; review devolves into taste debate | V5 + Acceptance Criterion 2: criteria must be observable on device or in code |
| Critical DES finding silently dropped | Audit problem survives the redesign | V8: traceability matrix must account for every finding, deferral requires rationale |

## Example Skeleton

```markdown
# IMPLEMENTATION Handover — NotesPro v3.2.0

## 1. Handover Header
| Field | Value |
|---|---|
| App | NotesPro (com.example.notespro) |
| App version | 3.2.0 (320) |
| Handover version | v1 |
| Producer | 01-core/prompts/08-implementation-handover.md |
| Date produced | 2026-06-12 |
| Factory version | 1.1.0 |
| Status | Delivered |
| Supersedes | — |
| Implements | DESIGN_HANDOVER_v2 + REDESIGN_PROPOSAL_v2 |

## 2. Design System Specification
| Role | Light | Dark | Notes |
|---|---|---|---|
| primary | #3D6B35 | #A3D49A | dynamic color overrides on API 31+ |

| Token | Compose target | XML target |
|---|---|---|
| primary | ColorScheme.primary — app/src/main/java/.../ui/theme/Color.kt | colorPrimary — app/src/main/res/values/themes.xml |

## 3. Component Specifications
### NoteCard
Anatomy: container (surfaceContainerLow), title (titleMedium/onSurface), ...
States: enabled, pressed (stateLayer 12%), selected (secondaryContainer). ...
Slots: title: String, preview: String, onClick: () -> Unit, selected: Boolean = false.

## 4. Motion Specification
| Context | Transition | Duration | Easing | Reduced-motion fallback |
|---|---|---|---|---|
| Note open (SCR-001→SCR-002) | container transform | 300ms | emphasized | instant; no shared-element animation when Remove animations is on |

## 5. Adaptive Layout Spec
Tablet/large-screen out of scope per DESIGN_HANDOVER_v2 §Adaptive & Device Context.

## 6. Ticket Breakdown
### IMPL-001 — Implement M3 theme (light + dark)
| Field | Value |
|---|---|
| Screen | Global |
| DES findings | DES-003, A11Y-001 |
| Files affected | app/src/main/java/.../ui/theme/Color.kt, Theme.kt, Type.kt |
| Spec | Section 2 of this document |
| Motion | — |
| A11y criteria | All role pairs meet WCAG AA 4.5:1; verified with Accessibility Scanner |
| Acceptance criteria | Theme compiles; debug screen renders all roles; dark switch verified on API 36 emulator |
| Effort | M |
| Dependencies | none |

### COPY-001 — Reword SCR-001 empty state
| Field | Value |
|---|---|
| Screen | SCR-001 |
| DES findings | — |
| Files affected | app/src/main/res/values/strings.xml (key empty_notes_title) |
| Spec | "No notes yet" → "Tap + to capture your first note" |
| Motion | — |
| A11y criteria | TalkBack announces new empty-state text |
| Acceptance criteria | String shows on empty list; no frozen string touched |
| Effort | S |
| Dependencies | none |

## 7. Implementation Sequence
| # | Ticket | Rationale |
|---|---|---|
| 1 | IMPL-001 | Theme-first: all screens restyle against tokens |
| 2 | COPY-001 | Copy change, after theme |

## 8. Asset Deliverables
| ID | Type | Spec | Format | Destination | Status |
|---|---|---|---|---|---|
| ASSET-001 | illustration | Empty-state, uses primary/tertiary | vector XML | app/src/main/res/drawable/ic_empty_notes.xml | Delivered |

## 9. Out of Scope
- DES-009 (Medium, onboarding rework) deferred to next cycle — tracked in PROJECT_STATE.md backlog.
- Proposed FAB bounce motion rejected (distracting; not in motion vocabulary).

## 10. Traceability Matrix
| Finding / change ID | Type | Severity | Resolved by | Or status |
|---|---|---|---|---|
| DES-003 | DES | Critical | IMPL-001, IMPL-004 | — |
| A11Y-001 | A11Y | High | IMPL-001 | — |
| Empty-state copy | COPY | — | COPY-001 | — |
```
