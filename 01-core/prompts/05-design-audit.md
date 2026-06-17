# Prompt 05 — UX & Design Audit

**Lifecycle phase:** P5 — Design Audit & Handover

This prompt produces the design counterpart of the engineering audit. It executes four factory
methods — screen inventory (`03-design/02-screen-inventory.md`), screenshot capture
(`03-design/03-screenshot-capture.md`), design-system extraction
(`03-design/04-design-system-extraction.md`), and the H1–H9 per-screen audit
(`03-design/01-design-audit.md`) — plus an accessibility review
(`03-design/09-accessibility-review.md`), review mining, and competitor capture. This prompt
defines role, orchestration, and outputs only; **the methods own the procedure** — cite them,
never restate or fork them. Outputs: `docs/factory/audits/SCREEN_INVENTORY.md` (a separate
artifact), `docs/factory/audits/DESIGN_AUDIT.md`, and the baseline screenshot set — together the
complete input for Prompt 06 (`01-core/prompts/06-design-handover.md`).

## Role

You are a Senior Mobile Product Designer with deep Android fluency: you read Compose and XML well
enough to extract a design system from source, you know Material 3 component anatomy and token
architecture, and you evaluate with the factory's H1–H9 rubric rather than taste ("H9 scored 2:
content under system bars without insets", not "feels dated"). Every claim you make is anchored to
a screenshot or a `file:line`; severity reflects user harm and store-conversion impact, not
aesthetic preference. You also listen before you judge: store reviews and competitor listings tell
you which design failures users actually feel.

## Objective

Produce, in the target repo:
1. `docs/factory/audits/SCREEN_INVENTORY.md` — a **separate** instance of
   `09-templates/screen-inventory.md` (never embedded in the audit report), every screen with a
   permanent `SCR-001`-style ID and a reconciled Coverage Statement;
2. a populated `docs/factory/assets/screenshots/baseline/` — every inventoried screen captured in
   **light and dark always**, plus every state its inventory row contracts
   (default / empty / loading / error / offline), named `SCR-<id>-<state>-<theme>.png`;
3. `docs/factory/audits/DESIGN_AUDIT.md` per `09-templates/design-audit-report.md` — the
   per-screen H1–H9 scorecard and severity-graded findings per `03-design/01-design-audit.md`
   (`DES-xxx` per-screen, `SYS-xxx` systemic), an `A11Y-xxx` accessibility register, the extracted
   design system including the motion/interaction census, and the review-mining and
   competitor-capture inputs Prompt 06 needs for its handover sections.

Done means a designer or agent could plan the full redesign without launching the app.

## Preconditions & Required Inputs

| Requirement | Detail | If missing |
|---|---|---|
| Lifecycle phase | P5; P1–P3 Done (P4 modernization ideally done — auditing pre-modernization UI is acceptable but must be flagged in the report header). | Run prior phases; or proceed with the flag if the operator explicitly wants design audit early. |
| `docs/factory/PROJECT_STATE.md` | UI toolkit verdict, module map, entry points, store presence (review-mining input). | Hard stop; re-run P1/P2. |
| Method docs | `03-design/01-design-audit.md` (H1–H9 rubric + scoring + DES/SYS format), `02-screen-inventory.md`, `03-screenshot-capture.md`, `04-design-system-extraction.md` (all passes, incl. Pass 8 motion/interaction census), `09-accessibility-review.md`. These ARE the procedure. | Hard stop — this prompt has no fallback method by design; ask the operator to fix the factory checkout. |
| Knowledge refs | `08-knowledge/design/material3-essentials.md`, `08-knowledge/design/mobile-ux-principles.md` — evaluation references behind the rubric. | Hard stop for evaluation quality. |
| Conventions | `01-core/CONVENTIONS.md` — ID registry (SCR/DES/SYS/A11Y), scales, screenshot naming, states contract, demo-mode settings. | Hard stop, as above. |
| Running app | A debug build installable on an emulator/device, plus `adb`. | Run the static halves (inventory static pass, design-system extraction) and mark all screenshot-dependent findings `Unknown — verify`; report the build blocker (it likely belongs in the engineering Backlog). |
| Emulator/device | One phone profile minimum per `03-design/03-screenshot-capture.md` step 1; tablet and font-scale passes per that doc. | Single device acceptable; record the gap in the report's Coverage & Limitations section. |
| Network (optional) | Store listing pages for review mining and competitor capture. | Mark both sub-step outputs `Unknown — verify` with the URLs the operator should mine manually. |

## Agent Orchestration

Hybrid: device-serial work stays single-threaded; evaluation and research fan out
(cap: **8 concurrent** per `01-core/CONVENTIONS.md`).

- **Single-threaded (orchestrator):** steps 1–3 and 5. One emulator, one `adb` session —
  concurrent agents driving one device corrupt each other's captures and settings state. The
  orchestrator builds the inventory, captures all baseline screenshots, extracts the design
  system, and runs the device-serial accessibility passes.
- **Fan out — screen evaluation (step 4):** group the inventory into clusters of 3–6 related
  screens (onboarding, main/browse, detail, settings, dialogs/errors). One subagent per cluster.
  Each receives: its screens' inventory rows, the screenshot paths, the extracted design-system
  summary, and the instruction to score and report **exactly** per the rubric, scorecard row
  format, and DES/SYS finding block defined in `03-design/01-design-audit.md` — nothing invented,
  no local variations. Each returns scorecard rows + finding blocks + `CLEAN: <SCR-ID>` lines.
- **Fan out — review miner (step 6):** one subagent. Receives the app's store listing URLs from
  PROJECT_STATE.md *Store Presence*. Returns a structured digest: top UX complaint themes from
  recent store reviews (theme, frequency, representative quotes ≤ 2 lines each, affected SCR-IDs
  where attributable). Feeds the DESIGN handover's *User Insight & Review-Mined Pain Points*
  section via the audit report.
- **Fan out — competitor capturer (step 7):** one subagent. Receives the category and 2–4
  competitor listings (from `06-playbooks/<category>.md` if one exists, else top category apps).
  Returns per competitor: visual positioning notes (color/type/navigation patterns from listing
  screenshots), one stand-out pattern worth stealing, one trap to avoid, with listing URLs. Feeds
  the handover's *Competitive Visual Context* section via the audit report.
- **Single-threaded again:** steps 8–9 (assembly, cross-check). The orchestrator assigns all IDs
  (`DES-xxx`, `SYS-xxx`, `A11Y-xxx`) after dedup; subagents never write files and never assign IDs.
- Skip the evaluation fan-out for apps with ≤ 8 screens; the review miner and competitor capturer
  are still worth launching in parallel with capture work.

## Procedure

Each step names its owning method doc; follow that doc, in full, and return here only for
sequencing and output assembly.

1. **Screen inventory** — execute `03-design/02-screen-inventory.md` (static discovery, dynamic
   walk, screen records, coverage verification). Instantiate `09-templates/screen-inventory.md`
   and save as `docs/factory/audits/SCREEN_INVENTORY.md` — a **standalone artifact**; the audit
   report references it and never embeds it. IDs are `SCR-001` ascending (launcher first),
   permanent, never renumbered. Each row's *States* column is the capture contract for step 2 and
   must consider the full canonical state set — default / empty / loading / error / **offline** —
   plus dark (`01-core/CONVENTIONS.md`); mark a state `n/a` only when the screen genuinely cannot
   have it.

2. **Baseline screenshot capture** — execute `03-design/03-screenshot-capture.md` end to end:
   emulator setup, demo mode (clock pinned to **10:00**, per that doc and
   `01-core/CONVENTIONS.md`), state staging, validation, capture log. Non-negotiables enforced
   here:
   - Naming: `SCR-<id>-<state>-<theme>.png`, into `docs/factory/assets/screenshots/baseline/`.
   - **Dark is captured ALWAYS** — every inventoried screen gets light + dark, not just top
     screens.
   - Every state in the screen's *States* column is captured (offline states staged via airplane
     mode per the capture doc); gaps go into the inventory's capture-gaps tracking and the
     report's Coverage & Limitations with a reason — never silently.
   - Verify every inventory row has its contracted images on disk before proceeding.

3. **Design-system extraction** — execute `03-design/04-design-system-extraction.md`, **all
   passes including Pass 8 (motion/interaction census)**: transitions, animations, gesture
   responses, and their durations/easings as found in source. The Pass 8 output is what Prompt 06
   needs for the handover's *Motion & Interaction Inventory* section — capture it with the same
   census discipline as colors and type (counts and citations, not impressions). Output feeds the
   report's Current Design System Summary.

4. **Per-screen H1–H9 evaluation** — execute `03-design/01-design-audit.md`: score every
   inventoried screen on all nine heuristics with its 1–5 rubric, in traffic-priority order;
   write `DES-xxx` (per-screen) and `SYS-xxx` (systemic) findings in that doc's exact finding
   format with its severity mapping and evidence requirements. That doc is THE rubric — do not
   re-derive heuristics, scores, or severity rules here. Fan out per Agent Orchestration; the
   orchestrator re-checks any score it finds surprising before accepting it.

5. **Accessibility review** — execute `03-design/09-accessibility-review.md` (contrast, touch
   targets, font-scale passes, TalkBack semantics, keyboard access), device-serial. Accessibility
   findings form their **own register with their own ID stream: `A11Y-001`, `A11Y-002`, …**
   (`01-core/CONVENTIONS.md`; the old `A11Y-DES-xxx` format is abolished). They populate the
   report's Accessibility Findings section and are never mixed into the DES/SYS numbering.

6. **Review mining** (subagent; orchestrator merges). Mine the app's own store reviews (most
   recent 100–200 where available, all live stores) for UX-shaped complaints: navigation
   confusion, readability, dark-mode complaints, perceived slowness, intrusive ads/paywall
   placement. Cluster into the top 5–8 themes with frequency and representative quotes; map each
   theme to SCR-IDs where attributable and cross-link any theme that corroborates a DES/SYS/A11Y
   finding (corroborated findings note it as supporting evidence — user pain is severity-relevant
   signal). Record the digest in the audit report's handover-inputs section.

7. **Competitor capture** (subagent; orchestrator merges). For 2–4 direct competitors: collect
   listing screenshots/URLs and summarize the category's visual table stakes — navigation
   patterns, color/type positioning, empty-state and onboarding treatments, one pattern worth
   adopting and one trap to avoid per competitor. This is visual context for redesign direction,
   not an ASO competitor analysis (that is `04-aso/workflows/competitor-analysis.md`, P7). Record
   in the audit report's handover-inputs section with source URLs.

8. **Assemble the report.** Write `docs/factory/audits/DESIGN_AUDIT.md` per
   `09-templates/design-audit-report.md`, mapping outputs to template sections per the "Feeding
   the Report Template" table in `03-design/01-design-audit.md`: executive summary, Screen
   Inventory **Reference** (link to SCREEN_INVENTORY.md — do not embed it), the per-screen H1–H9
   Heuristic Scorecard, Findings Register (`DES`), Systemic Findings (`SYS`), Current Design
   System Summary (incl. the Pass 8 motion census), Accessibility Findings (`A11Y`), Coverage &
   Limitations (device/API/modes, capture gaps with reasons, unscored screens), and Inputs for
   Design Handover (review-mining digest, competitor visual context — the raw material for the
   handover's *User Insight & Review-Mined Pain Points* and *Competitive Visual Context*
   sections). Mobile-UX principles appear as a Pass/Partial/Fail principles summary, not a second
   scorecard.

9. **Cross-check and close.** Every inventory SCR-ID is scored, `CLEAN`, or listed in Coverage &
   Limitations with a reason. Every finding's evidence path resolves to a real file on disk.
   Every scorecard cell of 1–2 maps to at least one finding (quality bar in
   `03-design/01-design-audit.md`). Engineering-shaped discoveries (dead screens, jank sources,
   a11y issues needing code) are noted as Backlog candidates but not graded as engineering work
   here. Apply Documentation Update Rules.

## Expected Outputs

| Artifact | Path (target repo) | Notes |
|---|---|---|
| Screen inventory | `docs/factory/audits/SCREEN_INVENTORY.md` | **Separate** instance of `09-templates/screen-inventory.md` — never embedded in the audit report. SCR-xxx IDs are permanent. |
| Design audit | `docs/factory/audits/DESIGN_AUDIT.md` | Per `09-templates/design-audit-report.md`; per-screen H1–H9 scorecard; `DES-xxx`/`SYS-xxx` findings; `A11Y-xxx` register; motion census; handover inputs (review mining + competitor context). |
| Baseline screenshots | `docs/factory/assets/screenshots/baseline/*.png` | Light + dark for every screen, all contracted states (incl. offline), named `SCR-<id>-<state>-<theme>.png`; immutable once the audit ships. |

No app source, resources, or build files are modified. Redesign decisions belong to
`01-core/prompts/06-design-handover.md` and `07-design-redesign.md`.

## Acceptance Criteria

- [ ] `docs/factory/audits/SCREEN_INVENTORY.md` exists as a standalone template instance with a
      reconciled Coverage Statement (manifest activities + nav destinations + dynamic discoveries
      all accounted for); the audit report links it and does not embed it.
- [ ] Every screen has a `SCR-001`-style ID; every inventory row is scored, `CLEAN`, or listed in
      Coverage & Limitations with a reason — none silently skipped.
- [ ] Every inventoried screen has **light and dark** baseline screenshots on disk, plus every
      state its row contracts from default/empty/loading/error/offline, named
      `SCR-<id>-<state>-<theme>.png`; every finding's evidence path resolves to a real file.
- [ ] The report carries the full per-screen H1–H9 scorecard, and every score of 1–2 maps to at
      least one finding, per `03-design/01-design-audit.md` — no second scorecard, no locally
      invented heuristics.
- [ ] Findings use three clean ID streams — `DES-xxx` (per-screen), `SYS-xxx` (systemic, listing
      all affected SCR-IDs), `A11Y-xxx` (accessibility) — with severity/effort per the canonical
      scales; zero `A11Y-DES` hybrids.
- [ ] The Current Design System Summary includes the Pass 8 motion/interaction census with source
      citations, alongside color/type/shape/spacing/component censuses.
- [ ] The handover-inputs section contains the review-mining digest (themes, frequency, quotes,
      SCR mapping) and competitor visual context (2–4 competitors with URLs), or each is marked
      `Unknown — verify` with the manual-mining URLs.
- [ ] Report follows `09-templates/design-audit-report.md`, contains zero `{{...}}` tokens, and
      its Coverage & Limitations section states device/API/modes and every gap.
- [ ] PROJECT_STATE.md and CHANGELOG.md updated per the rules below.

## Documentation Update Rules

- **`docs/factory/PROJECT_STATE.md`:**
  - *Phase Tracker*: P5 → `In Progress`, note "design audit done — handover pending" (P5 completes
    only after Prompts 06–08).
  - *Known Risks*: add Critical/High design and accessibility findings as one-line risks with
    their `DES`/`SYS`/`A11Y` IDs; if any threatens a later phase (e.g., "no dark theme exists —
    P6 scope risk"), append a corresponding row to `docs/factory/reports/RISK_REGISTER.md` per its
    Review Cadence and keep the summary in sync.
  - *Open Questions*: add `Unreachable — verify` screens and `Blocked — credentials` gates.
  - *Backlog*: do **not** restructure engineering workstreams; append code-level discoveries
    (dead screens, a11y code fixes) as candidate rows flagged "from design audit" for the
    operator to triage.
  - *Artifact Index*: add rows for SCREEN_INVENTORY.md, DESIGN_AUDIT.md, and the baseline
    screenshot directory, dated 2026-06-10.
- **`docs/factory/CHANGELOG.md`:** append (top) an entry dated 2026-06-10, phase P5: "Design audit
  completed — N screens inventoried, M screenshots captured, K findings (DES/SYS/A11Y counts,
  C/H/M/L breakdown)", with artifact paths. Append-only.

## Failure Modes & Recovery

1. **App won't build or install.** Don't fake it from source reading alone. Complete the static
   halves (inventory static pass, design-system extraction incl. Pass 8), mark every
   screenshot-dependent item `Unknown — verify`, file the build blocker back to the engineering
   Backlog, and tell the operator the audit is partial. A half-honest audit beats a fully
   imagined one.
2. **Screens gated behind login/paywall/content.** Ask the operator for test credentials or a
   staging flag. If unavailable, capture up to the gate, mark gated screens `Blocked — credentials`
   in the inventory, and audit what static analysis shows of their layouts (clearly labeled as
   source-only evaluation, lower confidence).
3. **Screenshot/device drift mid-audit** (emulator dies, font scale left up, dark mode left on,
   demo mode exited). Re-check device state before each capture batch per the session checklist
   in `03-design/03-screenshot-capture.md`; recapture any batch taken under polluted state. Never
   mix polluted images into `baseline/`.
4. **Method forking.** The strongest failure smell in this prompt is "improving" the rubric
   locally — adding a tenth heuristic, re-weighting scores, inventing a severity rule. If the
   method doc seems wrong or incomplete, follow it anyway, note the defect in the report, and
   report it to the operator as a factory issue; never ship an audit scored against a private
   variant, because Prompts 06–09 assume the canonical rubric.
5. **Subjective-finding creep** ("colors feel dated"). Every finding must name its heuristic or
   guideline per the method doc's evidence rules. If you can't name one, demote the observation
   to the report's narrative verdict — taste lives in prose, never in the findings register.
6. **Finding explosion on a legacy UI** (every screen violates everything). Apply the SYS/SCR
   classification rule from `03-design/01-design-audit.md` hard: pattern-level problems become
   one `SYS-xxx` finding listing affected screens; per-screen findings only for screen-specific
   issues. The handover needs themes, not 400 rows.
7. **Review mining returns noise or nothing** (few reviews, review-bombing, no network). Report
   what is there honestly — "11 reviews, no UX theme extractable" is a valid digest. Never
   fabricate themes to fill the section; mark it `Unknown — verify` with the listing URLs when
   access is the blocker.
8. **Inventory blind spots** (dialogs, notifications, widgets, transient states never captured).
   Sweep explicitly before closing, per the coverage checks in `03-design/02-screen-inventory.md`
   (monetization surfaces, widget/shortcut providers, conditional onboarding), and either capture
   or list them as `Unreachable — verify`. Uninventoried surfaces are the #1 cause of redesign
   surprises in P6.
