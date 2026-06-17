# Prompt 02 — Project Memory Initialization

**Lifecycle phase:** P2 — Project Memory

This prompt turns the raw reconnaissance from P1 into durable project memory. It creates the
`docs/factory/` tree inside the target repo, instantiates the factory's state and changelog
templates, and fills every placeholder token from `docs/factory/audits/DISCOVERY.md`. After this
prompt runs, every subsequent agent session on the project can cold-start from
`docs/factory/PROJECT_STATE.md` alone, and every phase records its history in
`docs/factory/CHANGELOG.md`.

## Role

You are a Technical Documentation Architect specializing in agent-operated knowledge bases. You
treat documentation as load-bearing infrastructure: tokens are filled with cited facts or marked
unknown — never guessed — because a wrong value in project memory silently corrupts every phase
that follows. You are precise about file structure, naming, and append-only history.

## Objective

Establish complete, accurate project memory in the target repo: `docs/factory/PROJECT_STATE.md`
instantiated from `01-core/PROJECT_STATE_TEMPLATE.md` with **zero** unfilled `{{...}}` tokens
remaining (every token either holds a fact cited from DISCOVERY.md or the literal value
`Unknown — verify`), `docs/factory/CHANGELOG.md` instantiated from `01-core/CHANGELOG_TEMPLATE.md`
with an onboarding entry, the standard subdirectory tree created per the canonical paths in
`01-core/CONVENTIONS.md`, and the Phase Tracker showing P1 Done and P2 Done after self-verification
passes. Done means a fresh agent reading only PROJECT_STATE.md can correctly state what the project
is, where it stands in the lifecycle, and what happens next.

## Preconditions & Required Inputs

| Requirement | Detail | If missing |
|---|---|---|
| Lifecycle phase | P2; P1 must be complete. | Run `01-core/prompts/01-discovery.md` first. Do not improvise discovery facts here. |
| `docs/factory/audits/DISCOVERY.md` | The sole source of project facts for token filling. | Hard stop. Report to operator: "P1 artifact missing." |
| `01-core/PROJECT_STATE_TEMPLATE.md` | Factory template for project memory; its Field Guide is the authoritative token list. | Hard stop — factory checkout is broken; ask operator to fix the factory path. |
| `01-core/CHANGELOG_TEMPLATE.md` | Factory template for the changelog. | Hard stop, as above. |
| Write access to target repo | You will create `docs/factory/**` and optionally a root `CLAUDE.md`. | Ask operator to grant write access. |
| Tools | File read/write/glob. No build execution, no network needed. | N/A |

Before writing anything, check whether `docs/factory/PROJECT_STATE.md` already exists. If it does,
this is a **re-onboarding**, not a fresh init — see Failure Modes item 2; never blindly overwrite
live project memory.

## Agent Orchestration

**Stay single-threaded.** This prompt is a deterministic transform of one input file
(DISCOVERY.md) through two templates into a small file tree. There is no parallelizable work, and
fan-out would risk two agents writing the same file. The only acceptable delegation: if
DISCOVERY.md is missing values that a quick file check could resolve (e.g., a version the P1 agent
marked `Unknown — verify` but that sits in `gradle/libs.versions.toml`), you may launch **one**
short-lived subagent to read the specific target file and return the cited value — or simply read
it yourself. Do not re-run full discovery from inside P2.

## Procedure

1. **Read inputs fully.** Read `docs/factory/audits/DISCOVERY.md` end to end. Read
   `01-core/PROJECT_STATE_TEMPLATE.md` and `01-core/CHANGELOG_TEMPLATE.md` from the factory,
   including their Field Guide tables, which define every token you must fill.

2. **Inventory the tokens.** List every `{{SNAKE_CASE}}` token **from the template itself** — the
   template's Field Guide, not this prompt, is the authoritative vocabulary. For each token, locate
   its value in DISCOVERY.md and note the DISCOVERY section it came from. Build a
   token → value → source map *before* writing, so gaps are visible up front. Example map rows
   (tokens shown exist in the template):

   | Token | Value | Source |
   |---|---|---|
   | `{{APP_NAME}}` | PDF Reader Pro | DISCOVERY.md › Summary |
   | `{{PACKAGE_ID}}` | com.example.pdfreader | DISCOVERY.md › Manifest Surface (app/build.gradle.kts:12) |
   | `{{GRADLE_VERSION}}` | 8.13 | DISCOVERY.md › Build System (gradle/wrapper/gradle-wrapper.properties:5) |
   | `{{STORES_PUBLISHED}}` | Google Play | DISCOVERY.md › Store Presence |

   Note: the Tech Stack Snapshot includes a **CI** row — fill it from DISCOVERY.md › Build
   Variants, Signing & CI (`Unknown — verify` if P1 found no CI evidence; `None` if P1 positively
   established there is no CI). There is no separate CI token outside that table.

   Keep the map in your working notes; it is your fill checklist and your audit trail if a value
   is later challenged.

3. **Apply the unknown-value rule.** For any token whose value DISCOVERY.md does not establish:
   - First, attempt a targeted verification: if the value lives in one obvious file
     (version catalog, manifest, wrapper properties), read that file and cite it.
   - If still unresolved, fill the token with the literal string `Unknown — verify` and add a
     matching line to the PROJECT_STATE.md *Open Questions* section.
   - **Never guess, never extrapolate, never copy a "typical" value.** A blank-but-honest memory
     is recoverable; a confidently wrong one is not.

4. **Create the tree** (canonical paths: `01-core/CONVENTIONS.md`). In the target repo, create:

   ```text
   docs/factory/
   ├─ PROJECT_STATE.md          (step 5)
   ├─ CHANGELOG.md              (step 6)
   ├─ audits/                   (already exists from P1 — verify, do not recreate/overwrite)
   ├─ handovers/
   ├─ reports/
   └─ assets/
      ├─ screenshots/
      │  ├─ baseline/
      │  └─ redesigned/
      └─ store/                 (per-store/per-locale subdirs are created on demand in P7)
   ```

   ```powershell
   New-Item -ItemType Directory -Force docs\factory\handovers, docs\factory\reports,
     docs\factory\assets\screenshots\baseline, docs\factory\assets\screenshots\redesigned,
     docs\factory\assets\store
   ```

   ```bash
   mkdir -p docs/factory/{handovers,reports,assets/screenshots/{baseline,redesigned},assets/store}
   ```

   If the target repo's tooling ignores empty directories (git does), drop a one-line `.gitkeep`
   in each empty directory so the tree survives commits.

5. **Instantiate PROJECT_STATE.md.** Copy the full template structure to
   `docs/factory/PROJECT_STATE.md` and fill every token from your step-2 map. The template's
   canonical sections are: Header block, Current Status, Phase Tracker, App Identity, Tech Stack
   Snapshot, Architecture Summary, Module Map, Store Presence, Known Risks, Open Questions,
   Decision Log, Backlog, Artifact Index, Release History, Session Log. Specifically ensure:
   - **Header block & App Identity** — app name, `{{PACKAGE_ID}}`, repo path, stores published,
     category, value proposition, target user, monetization model — all from DISCOVERY.md with its
     citations carried over.
   - **Tech Stack Snapshot** — SDK levels / Kotlin / AGP / Gradle / JDK / UI toolkit / DI / key
     libraries / **CI**, each with the `path:line` evidence DISCOVERY.md cited. Where the snapshot
     deviates from the factory tech baseline in `08-knowledge/android/version-matrix.md`, do not
     editorialize — P3 (`01-core/prompts/03-modernization-audit.md`) owns that judgment.
   - **Phase Tracker** — the P1–P8 table per `01-core/PROJECT_LIFECYCLE.md`. Set P1 = `Done`
     (date from DISCOVERY.md, artifact `docs/factory/audits/DISCOVERY.md`), **P2 = `In Progress`**
     (it flips to `Done` only after step 8's self-verification passes), P3–P8 = `Not Started`.
   - **Known Risks** — fill `{{RISK_REGISTER_LINK}}` with the canonical path
     `docs/factory/reports/RISK_REGISTER.md` annotated "(created in P3 by
     `01-core/prompts/03-modernization-audit.md`)" — P2 does **not** create the risk register.
     Seed the summary table with risks DISCOVERY.md makes obvious (e.g., checked-in keystore,
     defunct repository), or a single `—` row if none.
   - **Open Questions** — copy the entire *Open Questions* section of DISCOVERY.md, plus every
     `Unknown — verify` token from step 3.
   - **Backlog** — initialize empty with the template's column structure
     (# / Item / Severity / Effort / Source phase / Status); P3/P4 will populate it.
   - **Artifact Index** — table (artifact, path, version, date, status) seeded with the
     DISCOVERY.md row first, then PROJECT_STATE.md and CHANGELOG.md:

     | Artifact | Path | Version | Date | Status |
     |---|---|---|---|---|
     | Discovery report | `docs/factory/audits/DISCOVERY.md` | — | <P1 date> | Final |
     | Project memory | `docs/factory/PROJECT_STATE.md` | — | 2026-06-10 | Living |
     | Project changelog | `docs/factory/CHANGELOG.md` | — | 2026-06-10 | Living |

   - **Release History** — initialize empty with the template's columns (`—` row); P8 populates it.

6. **Instantiate CHANGELOG.md.** Copy `01-core/CHANGELOG_TEMPLATE.md` structure to
   `docs/factory/CHANGELOG.md`, fill its header tokens (project name, creation date 2026-06-10),
   and write the first entry — the onboarding entry:

   ```markdown
   ## 2026-06-10 — P1+P2 — Project onboarded into android-product-factory
   - Ran Prompt 01 (Discovery): produced docs/factory/audits/DISCOVERY.md.
   - Ran Prompt 02 (Project Memory): created docs/factory/ tree, instantiated
     PROJECT_STATE.md and CHANGELOG.md from factory templates.
   - Known gaps carried into Open Questions: <count> items.
   - Next: P3 — Engineering Modernization Audit (01-core/prompts/03-modernization-audit.md).
   ```

   Adjust the date line if P1 ran on a different day (two entries, one per phase, is also
   acceptable — newest on top, per the template's ordering rule).

7. **Optional root pointer (`CLAUDE.md`).** If the target repo root has no `CLAUDE.md`, create a
   short one (≤ 25 lines) so any future Claude session finds the factory memory immediately.
   Recommended shape:

   ```markdown
   # <App Name>

   <One-line description from PROJECT_STATE.md.>

   ## Factory project memory
   - This project is operated with the android-product-factory framework.
   - **Read `docs/factory/PROJECT_STATE.md` first** in every session — it holds the tech stack
     snapshot, phase tracker, and backlog.
   - History lives in `docs/factory/CHANGELOG.md` — append entries, never rewrite.
   - Operating rules: see `01-core/CLAUDE_MASTER.md` in the factory repo.
   ```

   If a `CLAUDE.md` already exists, **append** the "Factory project memory" section instead of
   replacing the file, and only after confirming the section is not already present.

8. **Self-verify, then close P2.** Run the checks mechanically, not by eyeball:

   ```powershell
   # Must return nothing — zero unfilled tokens:
   Select-String -Path docs\factory\PROJECT_STATE.md, docs\factory\CHANGELOG.md -Pattern '\{\{[A-Z0-9_]+\}\}'
   # Tree exists:
   Test-Path docs\factory\handovers, docs\factory\reports,
     docs\factory\assets\screenshots\baseline, docs\factory\assets\screenshots\redesigned,
     docs\factory\assets\store
   # Unknowns are mirrored — counts should correspond:
   (Select-String -Path docs\factory\PROJECT_STATE.md -Pattern 'Unknown — verify').Count
   ```

   ```bash
   grep -nE '\{\{[A-Z0-9_]+\}\}' docs/factory/PROJECT_STATE.md docs/factory/CHANGELOG.md  # expect no output
   ls docs/factory/handovers docs/factory/reports docs/factory/assets/screenshots/baseline \
      docs/factory/assets/screenshots/redesigned docs/factory/assets/store
   grep -c 'Unknown — verify' docs/factory/PROJECT_STATE.md
   ```

   Confirm every `Unknown — verify` in the body also appears in Open Questions, and that the
   Artifact Index lists DISCOVERY.md, PROJECT_STATE.md, and CHANGELOG.md. **Only after every check
   passes**, flip the Phase Tracker's P2 row from `In Progress` to `Done` (date 2026-06-10,
   artifact `docs/factory/PROJECT_STATE.md`) — the final state is P1 Done, P2 Done, P3–P8
   Not Started. If any check fails, fix and re-verify; P2 stays `In Progress` until then.

9. **Report.** Tell the operator: files created, token count filled vs marked unknown, and that
   the project is ready for P3.

## Expected Outputs

All paths are in the **target repo**:

| Artifact | Path | Notes |
|---|---|---|
| Project memory | `docs/factory/PROJECT_STATE.md` | Instantiated from `01-core/PROJECT_STATE_TEMPLATE.md`; zero tokens remain; Artifact Index seeded with the DISCOVERY.md row. |
| Project changelog | `docs/factory/CHANGELOG.md` | Instantiated from `01-core/CHANGELOG_TEMPLATE.md`; onboarding entry present. |
| Handover dir | `docs/factory/handovers/` (+ `.gitkeep`) | Empty; consumed from P5 onward. |
| Reports dir | `docs/factory/reports/` (+ `.gitkeep`) | Empty; first populated by P3 (incl. `RISK_REGISTER.md`, which P3 — not this prompt — creates). |
| Assets dirs | `docs/factory/assets/screenshots/baseline/`, `…/screenshots/redesigned/`, `…/store/` (+ `.gitkeep`) | Empty; populated in P5–P7 per the asset tree in `01-core/CONVENTIONS.md`. |
| Root pointer (optional) | `CLAUDE.md` in target repo root | Created or appended per step 7. |

Pre-existing and untouched: `docs/factory/audits/DISCOVERY.md`.

## Acceptance Criteria

- [ ] `docs/factory/PROJECT_STATE.md` exists and contains **zero** `{{...}}` placeholder tokens
      (verified by grep, not by eyeball).
- [ ] Every value in PROJECT_STATE.md traces to DISCOVERY.md (or a directly cited file), or reads
      exactly `Unknown — verify` — no invented values anywhere.
- [ ] Every `Unknown — verify` value is mirrored in the *Open Questions* section.
- [ ] The Tech Stack Snapshot's CI row is filled (value, `None`, or `Unknown — verify`).
- [ ] Known Risks links the risk register at its canonical future path and does **not** create it.
- [ ] The Artifact Index contains rows for DISCOVERY.md (seeded first), PROJECT_STATE.md, and
      CHANGELOG.md, with the D-columns artifact / path / version / date / status.
- [ ] Phase Tracker shows P1 = Done and P2 = Done (2026-06-10) — P2 flipped to Done only after the
      step-8 self-verification passed — with P3 through P8 = Not Started.
- [ ] `docs/factory/CHANGELOG.md` exists with the onboarding entry as its newest entry and zero
      placeholder tokens.
- [ ] Directories `handovers/`, `reports/`, `assets/screenshots/baseline/`,
      `assets/screenshots/redesigned/`, `assets/store/` exist; `audits/` still contains the
      untouched DISCOVERY.md.
- [ ] If a root `CLAUDE.md` was created or extended, it points at `docs/factory/PROJECT_STATE.md`
      and did not destroy pre-existing content.
- [ ] No file outside `docs/factory/**` and the optional root `CLAUDE.md` was created or modified.

## Documentation Update Rules

This prompt *creates* the documents the rest of the lifecycle updates, so its rules are
constitutive rather than incremental:

- **PROJECT_STATE.md** — fully instantiated here (every section, per Procedure step 5). From P3
  onward, other prompts update individual sections; this prompt is the only one allowed to create
  the file. It must never be re-instantiated over a live copy (see Failure Modes item 2). Its P2
  Phase Tracker row follows the canonical rule: `In Progress` from first write, `Done` only after
  the step-8 self-verification passes.
- **CHANGELOG.md** — created here with the onboarding entry. Establish the contract all later
  phases follow: newest entry on top, entries are append-only, every entry carries date, phase tag,
  and artifact paths touched. History below the newest entry is never edited.
- **RISK_REGISTER.md** — explicitly **not** created here. `01-core/prompts/03-modernization-audit.md`
  instantiates it at P3 from `09-templates/risk-register.md`; this prompt only records its
  canonical path in Known Risks.
- If this is a re-onboarding of a project whose memory already existed, do not create — instead
  update PROJECT_STATE.md in place (refresh Tech Stack Snapshot and Artifact Index, set the Phase
  Tracker honestly from the existing CHANGELOG) and append a `P2 — Memory refreshed` changelog
  entry describing exactly which fields changed.

## Failure Modes & Recovery

1. **DISCOVERY.md is missing or visibly incomplete** (e.g., Build System section has uncited
   versions, Modules section empty). Do not paper over it. Stop and run/re-run
   `01-core/prompts/01-discovery.md`; P2 must transform facts, not manufacture them.
2. **`docs/factory/PROJECT_STATE.md` already exists.** Never overwrite — it may contain months of
   backlog and phase history. Switch to refresh mode: diff existing content against DISCOVERY.md,
   update only stale facts, preserve Phase Tracker and Backlog, and log the refresh in CHANGELOG.md.
3. **A template token has no corresponding fact and no obvious source file.** Resist the urge to
   fill it with a plausible value. `Unknown — verify` + an Open Questions line is the correct
   output; flag the count of unknowns in your final report so the operator can prioritize closing
   them before P3.
4. **Template token vocabulary drifted** (the template contains a token its Field Guide doesn't
   explain, or vice versa). The template on disk wins over any token list you remember: fill what
   the Field Guide defines, mark the orphan token `Unknown — verify`, and report the template
   defect to the operator referencing `01-core/PROJECT_STATE_TEMPLATE.md` — the factory template
   needs a fix, which is outside this prompt's write scope.
5. **Target repo already has a conflicting docs convention** (e.g., existing `docs/` managed by a
   docs generator that lints unknown files). Still create `docs/factory/` (it is the factory's
   reserved namespace per `01-core/PROJECT_LIFECYCLE.md`), but note the coexistence in Open
   Questions and avoid touching the repo's other docs tooling.
6. **Write fails midway** (permissions, OneDrive sync lock). Re-verify which files landed using
   the step-8 checks, finish the remainder, and only then write the CHANGELOG entry — the
   changelog must describe the completed state, not the attempt. P2 stays `In Progress` in the
   Phase Tracker until the full self-verification passes.
