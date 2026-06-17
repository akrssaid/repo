# 05-handover — Handover Schemas

This directory defines the contract layer for AI-to-AI workflows in the factory. A handover is a
document produced by one agent (the producer) and executed by another agent (the consumer) that
may be a different AI product — e.g. Claude Code handing off to Claude Design — or a later Claude
session with zero shared context. Because AI sessions share no memory, **the document IS the
interface**: if a fact is not in the handover, it does not exist for the consumer. These schemas
make that interface rigorous enough that a consumer can execute from the document alone.

## Why schemas exist

- **No shared memory.** The producer's reasoning, tool outputs, and conversation history vanish
  when its session ends. Anything the consumer needs must be serialized into the handover.
- **Cross-product boundaries.** A handover may cross from Claude Code to Claude Design (or back),
  or to a human operator. The schema is the only thing both sides are guaranteed to understand.
- **Mechanical validation.** Each schema defines Validation Rules that are checkable without
  judgment ("section present", "every row has fields A,B,C", "every referenced path exists"),
  so `07-checklists/handover-validation.md` can gate consumption deterministically.
- **Accountability.** Acceptance Criteria give the consumer an explicit bar for accepting or
  rejecting, and the Rejection Protocol turns a bad handover into a versioned, auditable fix
  loop instead of a silent failure.

## The five handover types

The factory governs five handover types. Each is bound to a lifecycle phase, a single producer
prompt, a defined consumer, a schema in this directory, and the
`07-checklists/handover-validation.md` gate that must pass before consumption.

| Type | Schema | Phase | Produced by | Consumed by | Gate |
|---|---|---|---|---|---|
| DESIGN | `05-handover/DESIGN_HANDOVER_SCHEMA.md` | P5 — Design Audit & Handover | `01-core/prompts/06-design-handover.md` | `01-core/prompts/07-design-redesign.md` (Claude Design) | `07-checklists/handover-validation.md` |
| REDESIGN_PROPOSAL | `05-handover/REDESIGN_PROPOSAL_SCHEMA.md` | P5 — Design round trip (design → engineering) | `01-core/prompts/07-design-redesign.md` (Claude Design) | `01-core/prompts/08-implementation-handover.md` | `07-checklists/handover-validation.md` |
| IMPLEMENTATION | `05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md` | P5 → P6 boundary | `01-core/prompts/08-implementation-handover.md` | `01-core/prompts/09-ui-implementation.md` | `07-checklists/handover-validation.md` |
| ASO | `05-handover/ASO_HANDOVER_SCHEMA.md` | P7 — ASO & Store Assets | `01-core/prompts/14-store-assets.md` | Human operator + release prompts (`15-final-verification.md`, `16-release-preparation.md`) | `07-checklists/handover-validation.md` |
| RELEASE | `05-handover/RELEASE_HANDOVER_SCHEMA.md` | P8 — Verification & Release | `01-core/prompts/16-release-preparation.md` | Human operator + next-cycle agents (next P1/P3 run reads it as prior-cycle context) | `07-checklists/handover-validation.md` |

The DESIGN → REDESIGN_PROPOSAL → IMPLEMENTATION trio forms the design round trip: engineering
describes the app and its constraints to design (DESIGN handover, prompt 06); Claude Design returns
a **REDESIGN_PROPOSAL** (prompt 07, governed by `REDESIGN_PROPOSAL_SCHEMA.md`); engineering converts
that proposal into executable tickets for itself (IMPLEMENTATION handover, prompt 08). ASO and
RELEASE handovers terminate at a human operator because store consoles and rollout decisions
require a human in the loop.

## Where handover instances live

Schemas live here in the factory. Instances live in the TARGET app repo:

```
docs/factory/handovers/<TYPE>_HANDOVER_v<N>.md     TYPE ∈ DESIGN | IMPLEMENTATION | ASO | RELEASE
docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md   (the design round-trip return artifact)
```

`<N>` starts at 1 and increments on every revision. Examples:
`docs/factory/handovers/DESIGN_HANDOVER_v1.md`, `docs/factory/handovers/REDESIGN_PROPOSAL_v2.md`,
`docs/factory/handovers/ASO_HANDOVER_v3.md`.

## The validation gate

No handover may be consumed until it passes `07-checklists/handover-validation.md`. The gate
governs all five types (DESIGN, REDESIGN_PROPOSAL, IMPLEMENTATION, ASO, RELEASE) and runs the
generic checks (header complete with exactly the fields the type's schema requires, status set,
no placeholder tokens, no dangling file references, ID grep over the full `01-core/CONVENTIONS.md`
registry) plus the type-specific Validation Rules from the relevant schema in this directory. The
producer runs the gate before marking the handover **Delivered**; the consumer re-runs it before
marking it **Accepted**. A handover that fails the gate is rejected, never patched in place by the
consumer. Post-delivery, a `## Validation Appendix` may be appended to record re-run evidence
without mutating the frozen body.

## Versioning and rejection protocol (summary)

Full rules appear in each schema; the invariants are:

1. **Status lifecycle**: Draft → Delivered → Accepted | Rejected (the canonical handover status
   scale in `01-core/CONVENTIONS.md`). Status is recorded in the Handover Header of the document.
2. **Accepted handovers are immutable** (per `01-core/CONVENTIONS.md` — immutable after **Accepted**,
   not "after seen"). The `## Rejection Log` and `## Validation Appendix` sections are the **only**
   post-delivery appendable sections; everything else is frozen at acceptance. Corrections, scope
   changes, or new findings require a new version `v<N+1>`.
3. **Rejection is documented in the artifact.** The consumer appends a `## Rejection Log` section
   to the rejected handover listing each rejection reason with the schema rule or acceptance
   criterion it violates. The rejected file is kept as the audit trail.
4. **The producer issues v(N+1).** The new version addresses every logged rejection reason and
   states in its header which version it supersedes. The old version's status remains Rejected.
5. **Only one Accepted version per type per cycle.** Downstream work references the accepted
   version explicitly (e.g. "implements DESIGN_HANDOVER_v2", "converts REDESIGN_PROPOSAL_v2").

Record every handover delivery, acceptance, and rejection in `docs/factory/PROJECT_STATE.md`
(Artifacts and Phase Status sections) and `docs/factory/CHANGELOG.md`.

## Authoring rules common to all handovers

- Self-contained: the consumer must be able to execute without asking a single question. Links to
  other `docs/factory/` artifacts are allowed for depth, but every decision-critical fact must be
  restated in the handover itself.
- Every referenced file path must exist in the target repo at delivery time.
- Use the canonical scales from `01-core/CONVENTIONS.md`: Severity Critical/High/Medium/Low,
  Effort S/M/L/XL, Status Not Started/In Progress/Blocked/Done/Deferred, and the PASS / PASS with
  waivers / FAIL verdict vocabulary (the word "CONDITIONAL" is abolished).
- Stable IDs come from the ID registry in `01-core/CONVENTIONS.md` (format `PREFIX-NNN`,
  zero-padded 3): screens `SCR-xxx`, design findings `DES-xxx`, accessibility findings `A11Y-xxx`
  (its own stream), implementation tickets `IMPL-xxx`, copy-change tickets `COPY-xxx`, assets
  `ASSET-xxx`, experiments `EXP-xxx`. Never renumber IDs across handover versions.
