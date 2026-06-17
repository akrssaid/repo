# 01-core — The Factory Kernel

This directory is the kernel of the android-product-factory: the operating rules, the lifecycle,
the conventions, the two per-project memory templates, and the sixteen executable phase prompts.
Everything else in the factory (`02-engineering/` … `10-releases/`) is a library this kernel
calls into.

## Files in this directory

| File | What it is |
|---|---|
| `CLAUDE_MASTER.md` | Master operating instructions for every session: roles, session bootstrap/end procedures, orchestration rules, escalation. Loaded first, in full, every session. Wins all conflicts except convention definitions. |
| `CONVENTIONS.md` | Single source of truth for IDs, scales, verdicts, git formats, canonical paths, the artifact-contract matrix, versioning, and template mechanics. Every other doc cites it instead of restating. |
| `PROJECT_LIFECYCLE.md` | Authoritative definition of phases P1–P8: entry criteria, gates, loop-backs, modular entry rules. |
| `PROJECT_STATE_TEMPLATE.md` | Template for `docs/factory/PROJECT_STATE.md` in the target repo — the project's single source of truth, instantiated in P2. |
| `CHANGELOG_TEMPLATE.md` | Template for `docs/factory/CHANGELOG.md` — the session-by-session audit trail, instantiated in P2. |
| `WORKED_EXAMPLE.md` | One fictional app ("NotaFast") traced P1→P8 at miniature scale, with conformant artifact excerpts. Teaches by example; doubles as a conformance reference. |
| `prompts/01-…16-…` | The sixteen executable phase prompts (index below). |

## The 16 prompts

| # | Title | Phase | Primary artifact (under target `docs/factory/`) |
|---|---|---|---|
| 01 | Project Discovery | P1 | `audits/DISCOVERY.md` |
| 02 | Project Memory Initialization | P2 | `PROJECT_STATE.md`, `CHANGELOG.md` |
| 03 | Engineering Modernization Audit | P3 | `audits/ENGINEERING_AUDIT.md`, `reports/RISK_REGISTER.md` |
| 04 | Modernization Roadmap | P3 | `reports/MODERNIZATION_ROADMAP.md` |
| 05 | UX & Design Audit | P5 | `audits/SCREEN_INVENTORY.md`, `audits/DESIGN_AUDIT.md` |
| 06 | Design Handover Generation | P5 | `handovers/DESIGN_HANDOVER_v<N>.md` |
| 07 | Redesign Proposal | P5 | `handovers/REDESIGN_PROPOSAL_v<N>.md` |
| 08 | Implementation Handover | P5 | `handovers/IMPLEMENTATION_HANDOVER_v<N>.md` |
| 09 | UI Implementation | P6 | implemented UI code + `assets/screenshots/redesigned/` |
| 10 | Launcher Icon Redesign | P6 | icon assets + `reports/ICON_SPEC.md` |
| 11 | Store Screenshot Generation | P6 | `assets/store/<store>/<locale>/` + `reports/SCREENSHOT_MANIFEST.md` |
| 12 | ASO Audit | P7 | `audits/ASO_AUDIT.md` |
| 13 | Keyword Research | P7 | `reports/KEYWORD_MAP.md` |
| 14 | Store Asset & Listing Generation | P7 | `reports/STORE_LISTING_<store>_<locale>.md`, `handovers/ASO_HANDOVER_v<N>.md` |
| 15 | Final Verification | P8 | `reports/VERIFICATION_REPORT_v<versionName>.md` |
| 16 | Release Preparation | P8 | `handovers/RELEASE_HANDOVER_v<N>.md`, `reports/RELEASE_REPORT_v<versionName>.md` |

P4 (Modernization) has no dedicated prompt: it executes `02-engineering/migrations/*` guides
under CLAUDE_MASTER rules. The full prompt → artifact → template → schema → gate matrix is in
`CONVENTIONS.md` Section 4.

## The 9-section prompt contract

Every prompt in `prompts/` has exactly these nine H2 sections, in this order. A prompt missing
one is a factory defect — with **one deliberate exception**: `07-design-redesign.md` is pasted
into Claude Design (an external tool with no factory access), so it is written as a single
self-contained brief plus a *For the operator* section. Its output still conforms to
`05-handover/REDESIGN_PROPOSAL_SCHEMA.md`.

1. **Role** — the hat the agent wears (per CLAUDE_MASTER Section 1) and its mindset.
2. **Objective** — the artifact and the definition of done, in one paragraph.
3. **Preconditions & Required Inputs** — what must exist before starting, and what to do if it
   doesn't.
4. **Agent Orchestration** — single-threaded vs fan-out rules (cap 8), subagent contracts.
5. **Procedure** — numbered, executable steps.
6. **Expected Outputs** — exact paths and required structure of every artifact written.
7. **Acceptance Criteria** — the checkable list that gates "done".
8. **Documentation Update Rules** — what to write into PROJECT_STATE.md and CHANGELOG.md.
9. **Failure Modes & Recovery** — the known ways this prompt goes wrong and what to do.

## Precedence on conflict

Kernel documents have disjoint jurisdictions; when documents appear to disagree, the owner of
the jurisdiction wins, and the conflict is recorded in the target app's
`docs/factory/CHANGELOG.md` so the factory can be fixed:

- **Operating rules** (roles, session procedure, orchestration, escalation): `CLAUDE_MASTER.md`.
- **Convention definitions** (IDs, scales, paths, names, formats): `CONVENTIONS.md`.
- **Phase semantics** (what a phase is, when it is done, loop-backs): `PROJECT_LIFECYCLE.md`.

Nothing in `02-…/`–`10-…/` may override any of the three; library docs cite the kernel, never
restate it.

## Reading order

**New agent (a Claude session starting work):**

1. `CLAUDE_MASTER.md` — in full, before anything else.
2. Target repo `docs/factory/PROJECT_STATE.md` + CHANGELOG tail (per the bootstrap procedure).
3. `PROJECT_LIFECYCLE.md` — the section for the current phase.
4. The current phase's prompt in `prompts/`, plus the factory docs it declares as inputs.
5. `CONVENTIONS.md` — consult as needed; never restate it.

**New human (operator or contributor):**

1. Repo-root `README.md`, then `HOW_TO_USE.md` — what the factory is and how to drive it.
2. `QUICK_REFERENCE.md` (repo root) — the one-page operator card.
3. `PROJECT_LIFECYCLE.md` — the P1–P8 model.
4. `WORKED_EXAMPLE.md` — see one app go end to end.
5. `CLAUDE_MASTER.md` and `CONVENTIONS.md` — the rules your agents are following.
