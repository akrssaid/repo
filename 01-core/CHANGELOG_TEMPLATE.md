# CHANGELOG_TEMPLATE — Per-App Session Changelog Template

This template is instantiated into each target Android app as `docs/factory/CHANGELOG.md` — the
session-by-session narrative record that complements `docs/factory/PROJECT_STATE.md` (the state
file holds *current truth*; the changelog holds *how we got here*). The format adapts
Keep-a-Changelog to AI work sessions: entries are keyed by session rather than by release, and
include decisions and handovers as first-class categories. Every session appends exactly one
entry during the Session End Procedure (`01-core/CLAUDE_MASTER.md`, Section 2a), and every
Session Bootstrap reads the last 30 lines (Section 2).

=== TEMPLATE START ===

# CHANGELOG — {{APP_NAME}}

All notable work on this app by android-product-factory sessions. Format adapted from
[Keep a Changelog](https://keepachangelog.com) for AI sessions: one entry per session, newest
first. Companion to `docs/factory/PROJECT_STATE.md`.

## [{{ENTRY_DATE}}] — Session {{SESSION_NUMBER}} — Phase {{PHASE_ID}}

**Role:** {{AGENT_ROLE}}

### {{CATEGORY}}
- {{BULLET}}

=== TEMPLATE END ===

## Entry format

```text
## [YYYY-MM-DD] — Session N — Phase PX

**Role:** <role from CLAUDE_MASTER Section 1, e.g. Principal Android Engineer (migrator)>

### Added        ← new code, files, assets, documents
### Changed      ← modifications to existing code, config, or documents
### Fixed        ← defect corrections (include root cause in the bullet)
### Removed      ← deletions (code, dependencies, assets, documents)
### Decided      ← decisions made this session (mirror operator decisions into the Decision Log)
### Experiment   ← experiment events by EXP-NNN id (started/concluded/rolled out); state lives in `docs/factory/reports/EXPERIMENT_LOG.md`, this is the session trace
### Handover     ← handover documents produced, revised, validated, or consumed
```

Include only the categories the session actually used, in the order above. Omit empty
categories entirely — no "Nothing this session" filler.

## Rules

1. **Newest first.** New entries go directly under the `# CHANGELOG` intro paragraph, above all
   previous entries. Existing entries are never edited; corrections are bullets in a later entry.
2. **One entry per session**, even for blocked or fruitless sessions (a `### Decided` or
   `### Changed` bullet recording the blocker and state-file update is the minimum).
3. **Every entry names the phase and the agent role** — the `Phase PX` in the heading and the
   `**Role:**` line are both mandatory.
4. **Every code-touching entry references commits.** Cite short SHAs inline at the end of the
   relevant bullet, e.g. `(commits a1b2c3d, e4f5a6b)`. A bullet describing code work with no
   commit reference is invalid; if work was intentionally left uncommitted, say so and why.
5. **Every handover entry links the handover file** by repo-relative path, e.g.
   `docs/factory/handovers/DESIGN_HANDOVER_v2.md`, and states its validation result against
   `07-checklists/handover-validation.md`.
6. **Session numbers are global and sequential** across the whole engagement (not per-phase).
   Find the previous entry's number and add one.
7. **Bullets are specific and verifiable**: name files, versions, screens, and stores. "Improved
   the build" is not an entry; "Upgraded AGP 7.4.2 → 8.9.1" is.
8. **End every entry with the verification line** when the session touched code:
   `Build verification: PASSED` or `FAILED — <cause>`, matching the result of
   `02-engineering/07-build-verification.md`. Documentation-only sessions omit the line.

## Anti-patterns (reject on sight)

| Anti-pattern | Why it breaks the system | Instead |
|---|---|---|
| Editing a past entry to "fix history" | The changelog is the audit trail; the next session can no longer trust what it read last time | Add a correcting bullet in the current session's entry |
| One mega-entry covering several sessions | Bootstrap reads the tail expecting the *last session*; merged entries hide where work actually stopped | One entry per session, even tiny ones |
| Status narration ("still working on P4") with no facts | Burns the 30-line bootstrap window without informing the next session | State what changed, what was decided, what is next |
| Commit references like "(various commits)" | Unverifiable; defeats rule 4 | List the short SHAs, or state why work is uncommitted |
| Duplicating PROJECT_STATE.md tables into entries | Two sources of truth drift; the state file owns current truth | Link or name the section you updated, summarize the delta only |

## Worked example 1 — P4 modernization session

```text
## [2026-06-12] — Session 7 — Phase P4

**Role:** Principal Android Engineer (migrator)

### Changed
- Upgraded Gradle wrapper 7.5 → 8.13 per `02-engineering/migrations/gradle-migration.md`;
  regenerated wrapper files (commit 3f9c21a, `build(gradle): P4 Gradle wrapper 7.5 to 8.13`).
- Upgraded AGP 7.4.2 → 8.9.1; replaced removed `android.enableJetifier` flag, set
  `android.nonTransitiveRClass=true` (commit 8b04d7e).
- Migrated all dependency coordinates from `app/build.gradle` ad-hoc strings to
  `gradle/libs.versions.toml` version catalog (commit c5e1f02).

### Fixed
- Build failure after AGP upgrade: `:app` missing `namespace` declaration (AGP 8 requirement);
  moved package from manifest to `android { namespace }` (commit 9d2ab44).

### Removed
- Dropped unused `jcenter()` repository from `settings.gradle` (commit 8b04d7e).

### Decided
- Deferred Kotlin 1.8.22 → 2.1.x migration to next session: K2 compiler flags conflict with the
  pinned kapt setup; kapt → KSP must land first. Roadmap order unchanged. Backlog item #4 updated.

Build verification: `02-engineering/07-build-verification.md` PASSED (assembleDebug, lint, test
green; installed and launched on emulator API 36). `07-checklists/engineering-modernization.md`
items 1–4 checked off (gate not yet closed). Tech Stack Snapshot updated in PROJECT_STATE.md.
```

## Worked example 2 — P6 implementation session

```text
## [2026-06-19] — Session 11 — Phase P6

**Role:** Principal Android Engineer (UI specialist)

### Added
- Implemented redesigned Reader screen (SCR-003) as `ReaderScreen.kt` Compose implementation
  per IMPLEMENTATION_HANDOVER_v2 tickets IMPL-004..IMPL-008: M3 top app bar, dynamic-color
  theme tokens, bottom action sheet replacing the legacy overflow menu (commits 4ac81be,
  71d3e9f, e.g. `feat(ui): P6 IMPL-004 reader top app bar`).
- Captured redesigned screenshots (`SCR-003-default-light.png`, `SCR-003-default-dark.png`,
  empty/loading/error/offline states) to `docs/factory/assets/screenshots/redesigned/`
  (commit 0b8e2c1).

### Changed
- Promoted shared `FactoryButton` and `EmptyState` composables into `:core-ui` per ticket
  IMPL-007 (commit 71d3e9f).
- Applied COPY-002: Reader empty-state copy per the handover's Localization & Content
  Constraints section (commit 4ac81be). No un-ticketed string changes.

### Fixed
- Edge-to-edge inset regression on the Reader screen with API 36 gesture nav: root cause was a
  hardcoded status-bar padding in the legacy XML host; replaced with
  `WindowInsets.safeDrawing` handling (commit b62fa90).

### Decided
- Ticket IMPL-008 (animated page-turn) deferred to backlog: requires a paid Lottie asset —
  budget escalation raised to operator, decision pending. Marked Deferred in the ticket status
  table.

### Handover
- Consumed `docs/factory/handovers/IMPLEMENTATION_HANDOVER_v2.md`; validated against
  `05-handover/IMPLEMENTATION_HANDOVER_SCHEMA.md` via `07-checklists/handover-validation.md`
  before implementation — PASSED.

Build verification: PASSED. `07-checklists/design-modernization.md` in progress (gate closes
with the last screen). Phase P6 remains In Progress (4 of 7 screens done); next action set in
PROJECT_STATE.md: implement Settings screen (tickets IMPL-009..IMPL-012).
```

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` | Human-readable app name; must match PROJECT_STATE.md | `Solar PDF Reader` |
| `{{ENTRY_DATE}}` | Session date, absolute, `YYYY-MM-DD` | `2026-06-12` |
| `{{SESSION_NUMBER}}` | Global sequential session number (previous entry + 1) | `7` |
| `{{PHASE_ID}}` | Lifecycle phase of the session, `PX` | `P4` |
| `{{AGENT_ROLE}}` | Role from CLAUDE_MASTER Section 1, including the parenthetical mindset where applicable | `Principal Android Engineer (migrator)` |
| `{{CATEGORY}}` | One of: Added, Changed, Fixed, Removed, Decided, Experiment, Handover (repeat the H3 block per category used) | `Changed` |
| `{{BULLET}}` | One specific, verifiable item; commit SHAs for code work, file links for handovers (repeat per item) | `Upgraded AGP 7.4.2 → 8.9.1 (commit 8b04d7e)` |

## Usage

1. **Who instantiates it**: Claude, executing `01-core/prompts/02-project-memory.md` during
   phase P2, in the same session that instantiates PROJECT_STATE.md.
2. **Where the instance lives**: `docs/factory/CHANGELOG.md` in the TARGET app repo.
3. **Instantiation mechanics**: copy the block between TEMPLATE START/END (without markers),
   replace the header token, and write the first real entry — Session 1 (or the current session
   number if P1 ran as a separate session; backfill a P1 entry from DISCOVERY.md evidence if so).
   No `{{...}}` tokens may remain; the example entry skeleton is replaced by the real first
   entry, not kept.
4. **Update cadence**: exactly one new entry per session, written during the Session End
   Procedure (CLAUDE_MASTER Section 2a, step 4) — after build verification, before the final
   message. Long sessions may draft the entry incrementally at checkpoints.
5. **Reading cadence**: every Session Bootstrap reads the last 30 lines (CLAUDE_MASTER
   Section 2, step 4). Keep entries dense so 30 lines reliably covers the latest session; if an
   entry must run long, lead with the most decision-relevant bullets.
6. **Consistency duty**: the entry's phase, the Phase Tracker, and Current Status in
   PROJECT_STATE.md must agree at session end; on conflict the state file is truth and the
   changelog gets a correcting bullet next session.
