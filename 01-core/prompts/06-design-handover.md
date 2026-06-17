# Prompt 06 — Design Handover Generation

**Lifecycle phase:** P5 — Design Audit & Handover

This prompt packages everything a design AI (Claude Design) or human designer needs to redesign the
target app **without any access to the codebase, the device, or the operator's memory**. It is the
single most leverage-critical document in the factory: every omission here becomes a wrong guess on
the design side, and every wrong guess becomes rework in P6. Treat the handover as a contract, not
a summary.

## Role

You are a **Senior Mobile Product Designer with ten years of Android shipping experience**, acting
as the bridge between the engineering/audit side and the design side. You have just finished (or
have full access to) the design audit, and you know exactly what a designer needs and — equally
important — what a designer must NOT be allowed to change. You write with the discipline of someone
who has been burned by ambiguous handovers: every constraint explicit, every screen evidenced,
every finding traceable.

## Objective

Produce `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md` in the target app repo (N starts at 1;
see `01-core/CONVENTIONS.md` for versioning) — a complete, self-contained design brief that
conforms to `05-handover/DESIGN_HANDOVER_SCHEMA.md`, passes the
`07-checklists/handover-validation.md` self-check with verdict **PASS** (or PASS with waivers,
each waiver justified), and satisfies this bar: **a designer with no repo access, no prior
conversation, and no ability to ask questions could produce a complete, implementable redesign
proposal from this document and its referenced Asset Manifest files alone.**

## Preconditions & Required Inputs

| Requirement | Source | If missing |
|---|---|---|
| Lifecycle phase | P5, after the design audit | Run `01-core/prompts/05-design-audit.md` first |
| `docs/factory/PROJECT_STATE.md` | P2 (`02-project-memory.md`) | STOP — run P2; identity/brand/monetization facts live here |
| `docs/factory/audits/DESIGN_AUDIT.md` | P5 design audit | STOP — handover without audit findings is guesswork |
| Screen inventory | `docs/factory/audits/SCREEN_INVENTORY.md` (per `03-design/02-screen-inventory.md`) | Build it now using that guide before continuing |
| Baseline screenshots | `docs/factory/assets/screenshots/baseline/`, named `SCR-<id>-<state>-<theme>.png` (per `03-design/03-screenshot-capture.md` and `01-core/CONVENTIONS.md`) | Capture now — a handover without screenshots fails validation |
| Extracted design system + motion census | The "Current Design System" section embedded in `docs/factory/audits/DESIGN_AUDIT.md`, produced per `03-design/04-design-system-extraction.md` (Passes 1–7 tokens, **Pass 8 motion/interaction census**) | Run the extraction now; if Pass 8 was skipped, run it before writing the Motion & Interaction Inventory |
| Review + competitive evidence | P5 audit outputs where the audit already mined reviews / captured competitors; otherwise gathered here (Orchestration subagents 4–5) | Gather now — these feed mandatory schema sections |
| Motion vocabulary | `08-knowledge/design/material3-essentials.md` motion section (duration scale, easing sets, transition-pattern decision table, reduced-motion rule) | STOP — the handover must embed it; the designer has no factory-repo access |
| Schema | `05-handover/DESIGN_HANDOVER_SCHEMA.md` (factory repo) | STOP — never freelance the structure |
| Validation checklist | `07-checklists/handover-validation.md` (factory repo) | STOP — the self-check is mandatory |
| Tools | Read access to target repo; emulator/device for screenshot/landscape gap-filling; Play listing access for reviews/competitors | — |

Do not start writing the handover until every row above is satisfied. A partial handover is worse
than a late one.

## Agent Orchestration

Stay **single-threaded for the handover document itself** — it must read as one coherent voice with
one consistent set of constraints, and parallel authors create contradictions.

Fan out **up to 5 read-only subagents** (factory cap is 8, per `01-core/CONVENTIONS.md`) for
evidence gathering, and only for inputs that are incomplete or stale:

1. **Token verifier** — re-reads `res/values/colors.xml`, `themes.xml`, `dimens.xml`, font
   declarations, and/or Compose `Theme.kt`/`Color.kt`/`Type.kt`; returns a table of actual current
   tokens (name, value, usage count) to confirm the extraction section of DESIGN_AUDIT.md is
   current.
2. **Screenshot gap-filler** — compares `docs/factory/audits/SCREEN_INVENTORY.md` against
   `docs/factory/assets/screenshots/baseline/`; captures missing screens/states (including the
   landscape captures required by the Adaptive & Device Context section) with canonical
   `SCR-<id>-<state>-<theme>.png` names; returns the completed file list.
3. **Constraint scout** — greps the codebase for monetization surfaces (ad view IDs, `AdView`,
   `MaxAdView`, paywall activities, billing entry points), toolkit boundaries (which screens are
   XML, which are Compose), and localization reality (`res/values-<locale>/` dirs,
   `supportsRtl`, locale config); returns per-screen constraint + locale tables.
4. **Review miner** — pulls the current store rating and the most recent + most helpful user
   reviews from the app's Play listing (and other stores where present); returns the rating and
   the top 10 UX-relevant complaints, each quoted verbatim with an observed frequency count.
   Skip if the P5 audit already produced this — reuse its output instead of re-mining.
5. **Competitor capturer** — for 3–5 direct competitors (from PROJECT_STATE.md Store Presence or
   the P5 audit), captures each competitor's home screen + one core-task screen, plus one
   shelf-size icon row (all icons at ~48px side by side); returns images + a one-line
   visual-language read per competitor. Skip if the P5 audit already captured this.

Each subagent returns structured Markdown tables and files only; you merge them. Never let a
subagent write to the handover file.

## Procedure

1. **Read the schema first.** Open `05-handover/DESIGN_HANDOVER_SCHEMA.md` and copy its required
   section skeleton into a scratch outline. Every step below fills one or more schema sections, in
   schema order; if the schema requires a section not covered here, fill it from the same sources —
   **the schema wins over this prompt on structure and section numbering**.

2. **Write the Handover Header.** Fill every field the schema's Handover Header requires (app
   identity, versions, producer, date, factory version, status lifecycle
   Draft → Delivered → Accepted | Rejected, supersedes). In the app-version field also record the
   **git commit hash of the audited build** — if the codebase moves between handover and
   implementation, prompt 08 uses it to detect drift.

3. **Assemble App Identity & Brand** from `docs/factory/PROJECT_STATE.md`: app name, package ID,
   category, one-sentence value proposition, target audience, tone of voice; brand elements that
   are fixed (logo, mandated colors, partner branding) vs. negotiable; store positioning facts
   (rating, top markets, key competitors). Record each item with its source
   ("PROJECT_STATE.md › App Identity") so the design side can trust provenance.

4. **Gather Brand Asset Files.** The designer cannot open the repo, so export and list the actual
   brand source material:
   - Logo and wordmark exports (highest-fidelity available: SVG/vector master if it exists,
     else the largest raster found).
   - Font files in use (`res/font/`, bundled assets) — name each family **with its license
     status** (bundled OFL/Apache, Google Fonts, commercial license present/absent/unknown;
     unknown is a flagged risk, not a blank).
   - Current launcher icon layers: foreground, background, monochrome drawables from
     `res/mipmap-anydpi-v26/` and the drawables they reference, plus the current store icon if
     recoverable.
   Copy exports under `docs/factory/assets/brand/` and reference every file from this section and
   the Asset Manifest.

5. **Build the Screen Inventory section.** For every screen in
   `docs/factory/audits/SCREEN_INVENTORY.md`, emit one row with all columns the schema requires —
   including the **`States present` column**: which of the canonical per-screen states
   (default / empty / loading / error / offline, per `01-core/CONVENTIONS.md`) exist in the app
   and which are captured.
   - Screen IDs are stable (`SCR-001`…) and reused verbatim by prompts 07–09 — never renumber.
   - Every row must reference existing files in `docs/factory/assets/screenshots/baseline/` with
     canonical `SCR-<id>-<state>-<theme>.png` names. Missing screenshot → capture now
     (orchestration subagent 2); never ship a row with an empty evidence cell.
   - Dark captures are **always** required for inventoried screens; an app with no dark theme gets
     that recorded as a finding, not an omission.

6. **Document the Current Design System (extracted tokens).** Source: the "Current Design System"
   section of `docs/factory/audits/DESIGN_AUDIT.md` (per `03-design/04-design-system-extraction.md`)
   reconciled against the token verifier's output. Present color (hex, themed vs hardcoded),
   typography (families, weights, sp sizes, styles), spacing & shape (dimens, radii, elevation),
   and components (Material version mix — M2/M3/AppCompat — plus custom views). This section
   describes **what exists today**, not what should exist — no editorializing here.

7. **Write the Motion & Interaction Inventory.** Source: extraction **Pass 8** (motion/interaction
   census). Document what the app does today: screen transitions, list/item animations, ripple and
   state-layer usage, hero/shared elements, animated icons, scroll behaviors, haptics, and any
   reduced-motion handling — per screen where it varies, with "none" recorded explicitly. Then
   **embed the M3 motion vocabulary** from `08-knowledge/design/material3-essentials.md` verbatim:
   the duration scale (50–600ms short/medium/long), standard vs emphasized easing sets, the
   container-transform / shared-axis / fade-through decision table, and the reduced-motion
   fallback rule. The designer quotes these tokens in the proposal's per-screen Motion lines —
   without this table embedded, every Motion line downstream is invented.

8. **Write the Competitive Visual Context section.** From the competitor capturer (or P5 audit
   output): for each of 3–5 direct competitors, the home-screen + core-task screenshots, plus the
   single shelf-size icon row image, and a one-line visual-language read per competitor
   ("Competitor A: M3 expressive, bold tonal surfaces, oversized type"). Store images under
   `docs/factory/assets/competitive/` and reference them in the Asset Manifest. State what the
   category conforms on and where differentiation is available — facts and reads only; the
   redesign positioning decision belongs to prompt 07.

9. **Write the User Insight & Review-Mined Pain Points section.** From the review miner (or P5
   audit output): current store rating (per store where listed), and the **top 10 UX complaints,
   each quoted verbatim with its observed frequency** (e.g. "'can't find export' — 14 of 120
   recent reviews"). Tag each complaint with the affected `SCR-` ID(s) where attributable and
   cross-reference matching `DES-` findings. If reviews are too few to rank ten complaints, list
   what exists and state the sample size — never pad with invented complaints.

10. **Import and prioritize Audit Findings.** From `docs/factory/audits/DESIGN_AUDIT.md`, copy
    every finding with its original ID (`DES-001`…, `SYS-`/`A11Y-` where present — formats per
    `01-core/CONVENTIONS.md`), severity (Critical/High/Medium/Low), affected screen IDs, and
    evidence reference. Sort Critical → Low. Do not paraphrase finding IDs or severities — the
    redesign proposal (prompt 07) must trace its decisions back to these exact IDs.

11. **State Redesign Goals & Non-Goals.** Derive goals from PROJECT_STATE.md objectives plus audit
    themes and review-mined pain points (e.g. "adopt Material 3 with dynamic color", "make the
    primary action reachable one-handed"). List non-goals with equal force ("no
    information-architecture changes", "no new features", "icon handled by prompt 10"). Each goal
    must be testable by looking at the eventual proposal; vague goals ("make it modern") fail
    validation.

12. **Write the Hard Constraints section.** This is the contract clause. Enumerate, per
    constraint, the rule, the reason, and the affected screens:
    - **Toolkit boundaries**: which screens must remain XML, which are Compose, which may migrate
      (decision comes from the P3/P4 roadmap, not from design preference).
    - **Monetization placements that cannot move**: ad slots (format, screen, position), paywall
      trigger points, purchase buttons — cite `08-knowledge/monetization/ads-patterns.md` for why
      placement changes are revenue-sensitive.
    - **Platform conventions**: edge-to-edge is enforced (target SDK 35+), system back behavior,
      predictive back, Material 3 as the design language baseline
      (`08-knowledge/design/material3-essentials.md`).
    - **Technical ceilings**: minSdk-driven limits (e.g. dynamic color requires API 31+ — specify
      the fallback expectation), existing navigation graph that must not change.
    - **Accessibility floor**: WCAG-derived minimums from `07-checklists/accessibility.md` (4.5:1
      body text contrast, 48dp touch targets) are non-negotiable inputs to any palette.

13. **Write the Adaptive & Device Context section.** From Play Console / analytics where
    available, else stated estimates with basis:
    - Form-factor distribution (phone / tablet / foldable %), and whether **tablet is in scope**
      for this redesign cycle (a binding declaration prompts 07–09 build on).
    - Current orientation support per screen (locked portrait? rotation handled?) and tablet
      layout status (dedicated `sw600dp`/window-size-class layouts vs stretched phone UI).
    - **At least one landscape capture for every traffic-High screen**, named
      `SCR-<id>-<state>-<theme>-land.png` under the baseline directory and listed in the Asset
      Manifest — the designer cannot reason about landscape from portrait shots.

14. **Write the Localization & Content Constraints section.**
    - Shipped locales (`res/values-<locale>/` reality, not aspiration) and store-listing locales.
    - RTL status: `android:supportsRtl` value, whether any RTL locale ships, and known RTL defects.
    - Text-expansion guidance for the designer (e.g. "German strings run ~30% longer; labels must
      survive 2-line wrap or ellipsize per spec").
    - **The frozen-strings declaration (D13 contract)**: list which user-facing strings are
      FROZEN (legal, store-mandated, deep-linked, analytics-bound) and which are redesignable.
      Prompt 07 may propose copy changes **only** for redesignable strings; prompt 08 turns
      approved proposals into `COPY-NNN` tickets. An empty declaration is invalid — "all strings
      redesignable except: …" is the minimum form.

15. **Write the Imagery & Illustration Direction section.** Inventory current imagery: onboarding
    art, empty-state illustrations, photography, decorative shapes — style, source/license, and
    what is reusable vs dated. State the constraint set for new imagery (brand-fixed elements,
    license budget: e.g. "no stock photography; vector illustrations tintable to the M3 scheme")
    so the proposal's imagery direction lands inside reality, not a moodboard.

16. **Write Per-Screen Briefs.** One brief per screen ID with the Directive (Keep / Fix / Rethink)
    and specifics the schema requires: purpose, what works today, what the audit flagged (finding
    IDs), review complaints that land on this screen, screen-specific constraints (from the
    constraint scout), and what success looks like. Keep each brief 5–12 lines; the designer
    designs, you scope.

17. **Compile the Asset Manifest.** A single table — path under `docs/factory/assets/` | type |
    what it shows | referenced by (section / screen ID) — covering baseline screenshots (including
    landscape captures), brand asset files, competitive captures, and anything else the document
    references. A manifest file no section references is bloat; a section referencing a file
    absent from the manifest is a validation failure. State explicitly how the files are delivered
    to the design side (e.g. zipped `assets/screenshots/baseline/` + `assets/brand/` +
    `assets/competitive/` attached alongside the handover text). The name is **Asset Manifest** —
    never "Assets Bundle".

18. **Self-validate.** Run every item in `07-checklists/handover-validation.md` plus the schema's
    own validation rules against the draft. Record the evidence as a pass/fail table in a
    `## Validation Appendix` section at the end of the document (the schemas whitelist Validation
    Appendix and Rejection Log as the only post-delivery-appendable sections). Verdict scale:
    PASS / PASS with waivers / FAIL, per `01-core/CONVENTIONS.md`. Fix every failure before
    delivery — "delivered with known failures" is not a state this prompt permits.

19. **Version and deliver.** Save as `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md`, set Status
    to Delivered, and record the step-18 verdict in the header. If a prior version exists, bump N,
    set Supersedes, and add a "Changes from v<N-1>" section — never overwrite a delivered
    handover; never edit one after it reaches Accepted.

## Expected Outputs

| Artifact | Path (target repo) |
|---|---|
| Design handover document | `docs/factory/handovers/DESIGN_HANDOVER_v<N>.md` |
| Baseline screenshots (complete set incl. dark + landscape for traffic-High) | `docs/factory/assets/screenshots/baseline/` |
| Brand asset exports (logo/wordmark, fonts + license status, current icon layers) | `docs/factory/assets/brand/` |
| Competitive captures (home + core-task per competitor, shelf icon row) | `docs/factory/assets/competitive/` |
| Validation Appendix (inside the handover) | `## Validation Appendix` section within the handover |

## Acceptance Criteria

- [ ] Handover conforms to every required section of `05-handover/DESIGN_HANDOVER_SCHEMA.md`, in
      schema order — including Brand Asset Files, Competitive Visual Context, User Insight &
      Review-Mined Pain Points, Motion & Interaction Inventory, Adaptive & Device Context,
      Localization & Content Constraints, and Imagery & Illustration Direction.
- [ ] Handover Header is complete per the schema's field list, including the git commit hash of
      the audited build and the final validation verdict (PASS / PASS with waivers).
- [ ] Every item in `07-checklists/handover-validation.md` passes; the evidence table is embedded
      in the `## Validation Appendix`.
- [ ] Every screen in the Screen Inventory has a stable `SCR-` ID, a toolkit designation, a filled
      `States present` column, and existing baseline screenshots referenced by exact canonical
      filename (dark always; landscape for every traffic-High screen).
- [ ] Every Critical and High finding from `docs/factory/audits/DESIGN_AUDIT.md` appears with its
      original ID and severity, unmodified.
- [ ] The Motion & Interaction Inventory covers the extraction Pass 8 census AND embeds the M3
      motion vocabulary tables (durations, easings, pattern decision table, reduced-motion rule).
- [ ] User insight section quotes the top UX complaints verbatim with frequency counts and the
      current rating; competitive section covers 3–5 competitors with captures, icon row, and
      one-line reads.
- [ ] Localization & Content Constraints contains a non-empty FROZEN vs redesignable strings
      declaration; Adaptive & Device Context declares tablet in/out of scope.
- [ ] The hard-constraints section explicitly covers: XML-vs-Compose boundaries per screen,
      monetization placements that cannot move, edge-to-edge/platform conventions, minSdk-driven
      limits, and the accessibility floor.
- [ ] Goals are testable; non-goals are stated; neither section is empty. Every per-screen brief
      cites the finding IDs that apply to it (or states "no findings").
- [ ] The Asset Manifest (never "Assets Bundle") lists every referenced file; every listed file
      exists on disk; zero references to repo internals a no-repo-access designer could not
      resolve.
- [ ] **The litmus test**: read the document as a stranger — a designer with no repo access could
      produce a complete redesign from this document alone. If you hesitate, it fails.

## Documentation Update Rules

In the **target repo**:

- `docs/factory/PROJECT_STATE.md`:
  - **Phase Tracker**: set the P5 row to `In Progress` (or `Done` if prompts 07–08 already
    complete), with an absolute date (e.g. 2026-06-10).
  - **Artifact Index**: add a row for `handovers/DESIGN_HANDOVER_v<N>.md` with version, date, and
    validation verdict.
  - **Decision Log**: record constraint decisions made during assembly (e.g. "SCR-004 stays XML
    until P4 backlog clears", "tablet out of scope this cycle", the frozen-strings declaration)
    with rationale.
- `docs/factory/CHANGELOG.md`: add an entry under today's date:
  `P5 | Created DESIGN_HANDOVER_v<N>.md (N screens, M findings, validation: PASS) | prompt 06`.

Never edit `docs/factory/audits/DESIGN_AUDIT.md` from this prompt — the handover quotes the audit;
it does not amend it. If the audit is wrong, reopen prompt 05 and re-version.

## Failure Modes & Recovery

1. **Screenshot set is incomplete or stale** (UI changed since capture). Symptom: handover
   describes a screen the screenshot contradicts. Recovery: re-run capture per
   `03-design/03-screenshot-capture.md` on the current build, replace files in place (canonical
   names), and diff the screen inventory before resuming — never paper over with prose
   descriptions.
2. **Constraints are discovered to be wrong after delivery** (e.g. a "fixed" ad slot was actually
   removable). Recovery: do not silently message the design side; issue
   `DESIGN_HANDOVER_v<N+1>.md` with a "Changes from v<N>" section and update the PROJECT_STATE.md
   Decision Log. Constraint drift without a versioned document is how handover systems rot.
3. **Token extraction conflicts with reality** (the DESIGN_AUDIT.md extraction section says one
   primary color, layouts hardcode another). Recovery: trust the code, not the report — re-run
   extraction per `03-design/04-design-system-extraction.md`, document the discrepancy as evidence
   under the relevant `DES-` finding, and fix the audit section (via prompt 05 re-versioning)
   before the handover cites it.
4. **Review mining yields nothing usable** (unlisted app, <20 reviews, or no console access).
   Recovery: never fabricate complaints. State the limitation in the User Insight section ("12
   reviews total; 3 UX-relevant, quoted below"), substitute the P5 audit's heuristic pain points
   as clearly-labeled internal hypotheses, and log the gap in PROJECT_STATE.md Open Questions.
5. **The handover balloons past usefulness** (40+ screens, hundreds of findings). Recovery: keep
   all screens in the inventory table, but write full per-screen briefs only for the top-traffic
   screens plus every screen with a Critical/High finding; group long-tail screens into pattern
   briefs ("all settings sub-pages follow SCR-009's brief"). Note the grouping rule explicitly so
   prompt 07 covers them.
6. **No design audit exists and pressure says skip it.** Recovery: refuse. Run
   `01-core/prompts/05-design-audit.md` first, even in abbreviated form — a handover without
   finding IDs breaks traceability for prompts 07, 08, and 09, and the whole chain degrades into
   opinion.
7. **Validation checklist failures near deadline.** Recovery: failures are blockers by definition.
   Fix them; if a check is genuinely inapplicable (e.g. landscape captures for a
   rotation-locked kiosk app), record it as a waiver with one-line justification in the
   `## Validation Appendix` — the verdict becomes PASS with waivers; silent omission is FAIL.
