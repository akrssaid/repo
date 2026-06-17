# REDESIGN_PROPOSAL Schema

This schema defines the contract for the **design round-trip return artifact**: the document
Claude Design produces by running `01-core/prompts/07-design-redesign.md` (lifecycle phase P5),
consuming an accepted DESIGN handover (`05-handover/DESIGN_HANDOVER_SCHEMA.md`) and returning a
complete redesign that `01-core/prompts/08-implementation-handover.md` converts **mechanically** —
without asking a single question — into an IMPLEMENTATION handover. It is the fifth governed
handover type (see `05-handover/README.md`). The producer (the design AI) has no codebase access;
the consumer (prompt 08) has the codebase but none of the design conversation. Every decision must
therefore arrive as an exact value, a complete per-screen spec, or a pointer to a file that exists.

## File Naming & Versioning

- Instance path: `docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md` in the TARGET app repo.
- `<N>` starts at 1; every revision bumps N. Never edit a proposal after its status reaches
  Accepted — issue v(N+1) instead. After acceptance only `## Rejection Log` and
  `## Validation Appendix` may be appended (per `01-core/CONVENTIONS.md`).
- Status lifecycle (recorded in the Header): Draft → Delivered → Accepted | Rejected (canonical
  scale in `01-core/CONVENTIONS.md`).
- The header must name the accepted DESIGN handover version this proposal answers
  (e.g. "Answers: DESIGN_HANDOVER_v2").
- Must pass `07-checklists/handover-validation.md` before prompt 08 begins conversion.

## Required Sections

The document must contain exactly these 9 H2 sections, in this order — they are prompt 07's
deliverable skeleton. A missing or empty section is an automatic validation failure. Two further
sections are **whitelisted but not required** and may be appended after delivery without breaking
immutability: `## Rejection Log` (consumer, on reject) and `## Validation Appendix` (gate re-run
evidence). No other section may be added.

### 1. Header

A key/value table with all of: App name, Package ID, App version (versionName/versionCode),
Proposal version (vN), Producer (design AI: `01-core/prompts/07-design-redesign.md`), Date produced
(absolute, e.g. 2026-06-11), Factory version (from `VERSION.md`), Status (Draft / Delivered /
Accepted / Rejected), Supersedes (vN-1 or "—"), Answers (the accepted DESIGN handover version this
proposal responds to).

### 2. Design System Specification

The complete target token set with exact values, so prompt 08 can lift them straight into the
IMPLEMENTATION handover's Section 2. Required subsections, each a table, none empty:

- **Color**: every role with a light hex AND a dark hex. Dark is mandatory and must be derived as a
  **tonal palette** (Material 3 tonal steps), not by ad-hoc darkening — state the source palette.
- **Type scale**: M3 style name, font family, size (sp), line height (sp), weight.
- **Shape**: scale step (extraSmall…extraLarge), corner radius (dp), example components.
- **Spacing**: token name, value (dp), usage rule.
- **Elevation**: level, dp / tonal surface, where used.
- **Motion**: the motion vocabulary this redesign uses — durations and easings **quoted from the
  DESIGN handover's Motion & Interaction Inventory** and `08-knowledge/android/material3-essentials.md`
  (duration scale 50–600ms short/medium/long; standard vs emphasized easing; container transform /
  shared axis / fade-through patterns; reduced-motion fallback rule). This is the palette the
  per-screen Motion lines draw from.

If the app drops dark theme, that must be an explicit decision stated here, never an omission.

### 3. Per-Screen Redesign Specs

One H3 subsection per **every** SCR-ID in the DESIGN handover's Screen Inventory (no screen may be
skipped). Each spec must contain:

- **Layout**: the new structure (regions, hierarchy, navigation surface).
- **Components**: which Section 2 components/tokens each region uses.
- **States**: the visual for every applicable state — at minimum **default, empty, loading, error,
  offline** (offline always addressed, per the states contract in `01-core/CONVENTIONS.md`).
- **Motion**: one or more Motion lines, each naming context · transition · **duration** · **easing**
  · **reduced-motion fallback**, drawn from the Section 2 Motion palette.
- **Adaptive**: behavior across window-size classes **when tablet/large screen is in scope** per the
  DESIGN handover's Adaptive & Device Context; otherwise the single line "tablet out of scope".
- **Before/after rationale**: what changed and why, tied to specific **DES-ID / A11Y-ID** finding(s)
  this screen resolves.

### 4. Copy Change Proposals

Every proposed user-facing string change, each mapped to a future **COPY-NNN** ticket (D13). A
change may only target a string the DESIGN handover's Localization & Content Constraints marks
**Redesignable** — never a Frozen string.

| Proposal ref | Screen | String key / surface | Before | After | Targets future COPY ticket | Redesignable? (must be Yes) |
|---|---|---|---|---|---|---|

If no copy changes are proposed, state "no copy changes proposed" explicitly.

### 5. Icon & Splash Direction

The direction for the app icon and splash so prompt 08 can route them correctly. Icon procedure is
owned by `03-design/06-icon-redesign.md` (brief recorded in `reports/ICON_SPEC.md`); splash is owned
by prompt 09's procedure (`03-design/07-splash-redesign.md`). This section gives the visual intent
(concept, mono-stroke direction, color), not a re-specification of those procedures, and states
whether each is in scope this cycle.

### 6. Open Questions Back to Engineering

Any blocking uncertainty the design AI could not resolve from the DESIGN handover alone. **This
section being non-empty is a signal, not a failure** — but every entry must be answerable by
engineering before prompt 08 converts. If there are none, state "none — proposal is complete".

| # | Question | Why it blocks | Affected SCR-IDs |
|---|---|---|---|

### 7. Traceability

A single matrix proving coverage. One row per **DES-ID and per A11Y-ID** in the DESIGN handover's
Audit Findings:

| Finding ID | Type (DES / A11Y) | Severity | Addressed by (SCR-id / spec) | Or status (Deferred + rationale) |
|---|---|---|---|---|

Every **Critical/High DES finding and every A11Y finding** must be either addressed (named to a
per-screen spec) or explicitly **Deferred with a rationale**. Silence is not deferral.

### 8. Asset Manifest

A single table listing every file the proposal references or asks engineering to produce
(illustrations, exported logos used, mockups): path (relative to target repo root), type, whether
it is **Delivered** (exists now) or **To produce** (and by which screen/spec), and which section
references it. (Same name as the DESIGN handover's Asset Manifest, per `01-core/CONVENTIONS.md`.)

### 9. Acceptance Self-Check

The producer's own confirmation, before delivery, that the mechanical Validation Rules below pass —
a short checklist with each rule marked pass. This is what lets prompt 08 trust the proposal is
convertible without questions.

## Required Deliverables

1. The proposal document at `docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md`.
2. Every asset marked **Delivered** in the Asset Manifest, present at its stated path.
3. Any external mockup/spec file a per-screen spec points to, present at the referenced path.

## Validation Rules (mechanically checkable)

The proposal fails validation if any rule below fails. Run as part of
`07-checklists/handover-validation.md`.

| # | Rule |
|---|---|
| V1 | All 9 required H2 sections present, in order, non-empty (only `## Rejection Log` and `## Validation Appendix` may appear additionally) |
| V2 | Header contains all ten fields; Status ∈ {Draft, Delivered, Accepted, Rejected}; Answers names a DESIGN handover version |
| V3 | Every SCR-ID in the DESIGN handover's Screen Inventory has a Per-Screen Redesign Spec subsection (full coverage, no screen skipped) |
| V4 | Every per-screen spec covers default/empty/loading/error/offline states (or marks a state "not applicable" with reason) |
| V5 | Every Motion line — in Section 2 and in every per-screen spec — has a duration, an easing, and a non-empty reduced-motion fallback |
| V6 | Design System Specification color tables include both a light and a dark value for every role (or an explicit dark-theme-dropped decision); dark is a tonal palette |
| V7 | Every Critical/High DES finding and every A11Y finding in the DESIGN handover appears in the Traceability matrix, each Addressed or Deferred-with-rationale |
| V8 | Every Copy Change Proposal targets a string the DESIGN handover marks Redesignable (never Frozen) and names a future COPY ticket |
| V9 | No unresolved references: every SCR-ID / DES-ID / A11Y-ID resolves to the DESIGN handover; every asset path appears in the Asset Manifest and (if Delivered) exists on disk |
| V10 | No placeholder tokens (`{{...}}`), no "TBD"/"TODO" strings anywhere in the document |

## Acceptance Criteria (consumer's bar)

Prompt 08 (the consumer) accepts only if all hold; otherwise rejects:

1. **The mechanical-conversion test**: prompt 08 can produce the IMPLEMENTATION handover from this
   document **without asking a single question** — every token, state, motion line, and copy change
   is a value or a resolvable pointer, not "as discussed".
2. The Design System Specification is complete enough to populate the IMPL handover's Section 2
   verbatim (no missing role, no "pick something close"; dark theme fully specified).
3. Every inventoried screen has a spec covering all its states including offline; every Motion line
   carries duration + easing + reduced-motion fallback.
4. Every Critical/High DES finding and every A11Y finding is addressed or explicitly deferred, and
   every copy change is ticketable against a redesignable string.
5. Open Questions, if any, are answered (or empty); none remain blocking at acceptance.
6. All Validation Rules V1–V10 pass on the consumer's re-run of
   `07-checklists/handover-validation.md`.

## Rejection Protocol

1. The consumer appends a `## Rejection Log` section to the proposal file (one of only two
   permitted post-delivery appends, the other being `## Validation Appendix`), one row per reason:

   | Date | Rejected by | Rule / criterion violated | Detail |
   |---|---|---|---|

2. The consumer sets Status to Rejected in the Header and stops conversion.
3. The producer (design AI) issues `REDESIGN_PROPOSAL_v<N+1>.md` addressing every logged reason,
   with Supersedes set to vN. The rejected file is never deleted or edited further.
4. Both events are recorded in `docs/factory/PROJECT_STATE.md` and `docs/factory/CHANGELOG.md`.

## Producer & Consumer Workflow

1. **Producer** (prompt 07 / Claude Design) drafts with Status: Draft, working only from the
   accepted DESIGN handover and its attached baseline screenshots. Every design decision becomes a
   token value, a per-screen spec, a Motion line, or a copy proposal — nothing remains only in prose.
2. Producer fills the Acceptance Self-Check and self-runs the Validation Rules V1–V10.
3. Producer sets Status: Delivered; delivery is recorded in `docs/factory/PROJECT_STATE.md` and
   `docs/factory/CHANGELOG.md`.
4. **Consumer** (prompt 08) re-runs `07-checklists/handover-validation.md`, then dry-runs the
   mechanical-conversion test before accepting.
5. On accept: consumer sets Status: Accepted and converts the proposal into
   `IMPLEMENTATION_HANDOVER_v<N>.md`, citing `REDESIGN_PROPOSAL_v<N>`. On reject: follow the
   Rejection Protocol above.

## Common Failure Modes

| Failure | Symptom downstream | Prevention |
|---|---|---|
| Motion lines without reduced-motion fallback | IMPL §Motion Specification incomplete; D14 V-rule fails | V5: every Motion line needs duration + easing + reduced-motion fallback |
| Dark theme "derive from light" | Engineering invents a dark palette the designer never approved | V6: dark mandatory, tonal palette stated |
| Offline state ignored | App ships a redesigned screen with no offline treatment | V4: offline addressed on every screen |
| Copy change against a frozen string | COPY ticket banned at prompt 08; rework | V8: copy proposals only against Redesignable strings |
| A screen left unspecified ("keep as-is" in prose) | Prompt 08 must guess; conversion stalls | V3: every SCR-ID gets a spec, even if minimal |

## Example Skeleton

```markdown
# REDESIGN_PROPOSAL — NotesPro v3.2.0

## 1. Header
| Field | Value |
|---|---|
| App | NotesPro (com.example.notespro) |
| App version | 3.2.0 (320) |
| Proposal version | v1 |
| Producer | 01-core/prompts/07-design-redesign.md |
| Date produced | 2026-06-11 |
| Factory version | 1.1.0 |
| Status | Delivered |
| Supersedes | — |
| Answers | DESIGN_HANDOVER_v2 |

## 2. Design System Specification
### Color (dark via tonal palette: brand green source, M3 tonal steps)
| Role | Light | Dark | Used in |
|---|---|---|---|
| primary | #3D6B35 | #A3D49A | FAB, toolbar |
### Type scale
| Style | Family | Size (sp) | Line height (sp) | Weight |
|---|---|---|---|---|
| titleLarge | Roboto | 22 | 28 | 500 |
### Shape
| Step | Radius (dp) | Components |
|---|---|---|
| medium | 12 | cards |
### Spacing
| Token | Value (dp) | Usage |
|---|---|---|
| space-2 | 8 | list item padding |
### Elevation
| Level | dp / surface | Used in |
|---|---|---|
| 1 | surfaceContainerLow | cards |
### Motion (quoted from DESIGN_HANDOVER_v2 §Motion & Interaction Inventory)
| Pattern | Duration | Easing | Reduced-motion fallback |
|---|---|---|---|
| container transform | 300ms (medium) | emphasized | instant; no shared-element animation |

## 3. Per-Screen Redesign Specs
### SCR-001 — Note list
Layout: top app bar + LazyColumn + bottom banner (fixed). Components: NoteCard, FAB.
States: default (cards); empty (illustration + "Tap + to capture"); loading (skeleton rows);
error (retry banner); offline (sync-paused chip).
Motion: list open → container transform, 300ms, emphasized, reduced-motion = instant.
Adaptive: tablet out of scope.
Before/after: raises body contrast to 4.5:1 (resolves DES-003); FAB to 56dp (resolves A11Y-001).

## 4. Copy Change Proposals
| Proposal ref | Screen | String key | Before | After | Targets COPY ticket | Redesignable? |
|---|---|---|---|---|---|---|
| CP-1 | SCR-001 | empty_notes_title | No notes yet | Tap + to capture your first note | COPY-001 | Yes |

## 5. Icon & Splash Direction
Icon: mono-stroke pencil-in-circle, brand green — in scope (brief to 03-design/06-icon-redesign.md).
Splash: themed icon on surface — in scope (prompt 09 / 03-design/07-splash-redesign.md).

## 6. Open Questions Back to Engineering
| # | Question | Why it blocks | Affected SCR-IDs |
|---|---|---|---|
| 1 | none — proposal is complete | — | — |

## 7. Traceability
| Finding ID | Type | Severity | Addressed by | Or status |
|---|---|---|---|---|
| DES-003 | DES | Critical | SCR-001 spec | — |
| A11Y-001 | A11Y | High | SCR-001 spec | — |

## 8. Asset Manifest
| Path | Type | Delivered / To produce | Referenced in |
|---|---|---|---|
| docs/factory/assets/redesign/empty-notes.svg | illustration | To produce (SCR-001) | §3, §8 |

## 9. Acceptance Self-Check
V1 pass · V2 pass · V3 pass (all SCR covered) · V4 pass · V5 pass · V6 pass · V7 pass · V8 pass ·
V9 pass · V10 pass.
```
