# Handover Validation Checklist

Universal gate run **before any handover is consumed** — and self-run by the producing agent
before delivery. Handovers (`docs/factory/handovers/<TYPE>_HANDOVER_v<N>.md`) are the contract
between lifecycle phases; a malformed or incomplete handover silently corrupts every downstream
phase. This checklist verifies structure against the type's schema in `05-handover/`, completeness
(no placeholders, no empty required tables), reference integrity (every path and ID resolves), and
executable quality. The consumer runs it even if the producer already passed it: trust the
artifact, not the producer's memory.

> **Run when:** (1) By the producer, immediately before declaring a handover delivered; (2) by the
> consumer, before consuming any handover or any new version of one.
> **Run by:** Producer agent (self-gate) and consumer agent (acceptance gate) — two separate runs,
> two separate reports.
> **Pass condition:** 100% of non-N/A items checked. Verdict Accept on PASS; Reject on any FAIL.
> Record the completed copy as
> `docs/factory/reports/handover-validation-<TYPE>-v<N>-YYYY-MM-DD.md`.

Commands run from the target app repo root. `<H>` = the handover file under validation.

## Structural

- [ ] Handover type identified and validated against the correct schema:

      | TYPE | Schema | Produced by | Consumed by |
      |---|---|---|---|
      | DESIGN | `05-handover/DESIGN_HANDOVER_SCHEMA.md` | P5 (`01-core/prompts/06-design-handover.md`) | P5 redesign / P6 |
      | REDESIGN_PROPOSAL | `05-handover/REDESIGN_PROPOSAL_SCHEMA.md` | P5 (`01-core/prompts/07-redesign-proposal.md`, pasted into Claude Design) | P5 (`08-implementation-handover.md`) |
      | IMPLEMENTATION | `05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md` | P5 (`01-core/prompts/08-implementation-handover.md`) | P6 |
      | ASO | `05-handover/ASO_HANDOVER_SCHEMA.md` | P7 (`01-core/prompts/14-store-assets.md`) | P8 |
      | RELEASE | `05-handover/RELEASE_HANDOVER_SCHEMA.md` | P8 (`01-core/prompts/16-release-preparation.md`) | Human owner / next cycle |

      Verify: filename matches the artifact's naming (`<TYPE>_HANDOVER_v<N>.md`, or
      `REDESIGN_PROPOSAL_v<N>.md` for that type); open the schema.
- [ ] All schema-required H2 sections are present, in schema order, none renamed. Extra sections
      are permitted only where the schema whitelists them — `## Validation Appendix` and
      `## Rejection Log` are always permitted post-delivery appendable sections; any other extra
      section is a FAIL. Verify: `rg "^## " <H>` and diff the list against the schema's
      required-sections list — record any missing section, or any extra section not on the
      whitelist.
- [ ] Header block complete per the schema: **every header field the type's schema requires** has
      a non-empty value (consult the schema's header definition — do not assume a fixed field set).
      Verify: read the schema's header section, then confirm each required field is present and
      non-empty in `<H>`.
- [ ] Version number is correct: v1 for first delivery, or incremented by exactly 1 from the
      highest existing `<TYPE>_HANDOVER_v*.md`, with a "Changes from previous version" note when
      N > 1. Verify: `Get-ChildItem docs/factory/handovers/<TYPE>_HANDOVER_v*.md` and compare.

## Completeness

- [ ] No empty required tables: every table the schema marks required has at least one data row
      (header + separator alone = empty = FAIL). Verify: visual scan of each required table.
- [ ] No placeholder or deferred-content text anywhere. Verify:
      `rg -i "TBD|TODO|FIXME|placeholder|fill in|to be determined|\{\{" <H>` returns zero hits
      (template tokens `{{...}}` must all be instantiated).
- [ ] Coverage rule of the type satisfied — every inventory item is covered: DESIGN covers every
      screen in the screen inventory (SCR-xx) with a spec or an explicit out-of-scope entry;
      REDESIGN_PROPOSAL covers every in-scope DESIGN screen with a per-screen proposal (incl. its
      Motion line); IMPLEMENTATION covers every DES item with an IMPL ticket and every proposal
      Motion line maps to a ticket or is explicitly rejected; ASO covers every store × locale in
      scope; RELEASE covers every artifact in `05-handover/RELEASE_HANDOVER_SCHEMA.md`'s artifact
      list. Verify: count inventory items vs covered items; record both numbers.
- [ ] All statuses use the canonical scale (Not Started / In Progress / Blocked / Done /
      Deferred) and severities/efforts use the canonical scales where the schema requires them.
      Verify: scan status columns; no ad-hoc values like "WIP" or "done-ish".

## References

- [ ] Every cross-referenced file path in the handover exists. Verify: extract paths with
      `rg -o "docs/factory/[A-Za-z0-9_./\\-]+" <H> | sort -u` then test each
      (PowerShell: `Get-Content paths.txt | ForEach-Object { if (-not (Test-Path $_)) { $_ } }`
      must print nothing).
- [ ] Every ID reference resolves to a defined item — defined either in this handover or in the
      referenced upstream artifact. Verify across the full ID registry (`01-core/CONVENTIONS.md`):
      `rg -o "(SCR|DES|SYS|A11Y|ENG|ASO|IMPL|COPY|ASSET|WS|RISK|KW|EXP)-[0-9A-Z]+" <H> | sort -u`
      and check each ID against its definition table; record orphaned IDs. (Note `WS-A`-style and
      `A11Y-001`-style IDs are in scope.)
- [ ] No dangling forward references: nothing cites a section, appendix, or version that does not
      exist. Verify: check each "see section/appendix X" mention.
- [ ] Traceability matrix complete (IMPLEMENTATION type only; N/A for others with that
      justification): every IMPL ticket maps to its DES source and its SCR screen, no unmapped
      rows in either direction. Verify: matrix row count = IMPL ticket count; no blank cells.

## Quality

- [ ] Every ticket/section that the schema requires acceptance criteria for has them, and each
      criterion is objectively verifiable (a command, an observable state, a comparable value —
      same standard as this checklist). Verify: scan each ticket; flag any criterion that cannot
      fail (e.g. "looks good", "works well").
- [ ] Consumer-readiness question answered YES in writing: "Could an agent with zero prior
      context execute this handover without asking the producer a single question?" Verify: the
      consumer agent (or producer simulating one) reads the handover cold and records the answer
      plus any questions that arose — one unanswerable question = FAIL.
- [ ] Quantities and values are concrete: dimensions in px/dp, colors as tokens or hex, counts as
      numbers, dates absolute. Verify: spot-check specs for vague quantifiers ("large", "soon",
      "several").

## Disposition

- [ ] Verdict recorded: **Accept** (all items pass — consumer may proceed) or **Reject**.
- [ ] On Reject: each failed item logged as a rejection reason in the handover's Rejection Log
      section, following the rejection protocol in `05-handover/README.md` (reason, date,
      rejecting agent). The producer fixes and delivers `v<N+1>` — never edits the rejected
      version in place — and the new version is re-validated in full.
- [ ] This report saved to
      `docs/factory/reports/handover-validation-<TYPE>-v<N>-YYYY-MM-DD.md` and the run logged in
      `docs/factory/CHANGELOG.md` with the verdict.

## Sign-off

| Field | Value |
|---|---|
| Date | |
| Handover validated | `<TYPE>_HANDOVER_v<N>.md` |
| Run role | Producer self-gate / Consumer acceptance |
| Verdict | PASS (Accept) / FAIL (Reject) |
| Failed items | (list item text or "none") |
| N/A items + justification | |
| Run by | (agent/session identifier) |
