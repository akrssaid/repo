# VERSION

This file records the current version of the Android Product Factory framework and defines how the
framework itself is versioned, how target projects stay compatible with it, and how framework
releases are cut. The framework's full change history lives in
`10-releases/FRAMEWORK_CHANGELOG.md`; this file always reflects only the current release.

## Current Version

| Field | Value |
|---|---|
| Framework version | **1.1.0** |
| Released | 2026-06-10 |
| Status | Stable |
| Changelog | `10-releases/FRAMEWORK_CHANGELOG.md` |

## Versioning Strategy

The framework uses semantic versioning (`MAJOR.MINOR.PATCH`) adapted to a documentation framework.
The "public API" of this framework is the set of contracts that target-project documents depend
on: the conventions in `01-core/CONVENTIONS.md` (IDs, scales, naming, canonical paths), the
schemas in `05-handover/`, the lifecycle phase definitions in `01-core/PROJECT_LIFECYCLE.md`, and
the structure of `01-core/PROJECT_STATE_TEMPLATE.md` and `01-core/CHANGELOG_TEMPLATE.md`.

| Component | Bumped when | Examples |
|---|---|---|
| **MAJOR** | A breaking change to conventions, schemas, the lifecycle, or state/changelog template structure that invalidates documents already **consumed or Accepted** in target projects | Schema section removed or renamed in `05-handover/`; phase split/merged in `01-core/PROJECT_LIFECYCLE.md`; ID format or canonical path changed in `01-core/CONVENTIONS.md` |
| **MINOR** | Backward-compatible additions: new prompts, workflows, playbooks, checklists, templates, or knowledge files; schema sections added that affect only *future* (and in-flight, via re-validation) documents | New playbook in `06-playbooks/`; new store guide in `04-aso/stores/`; new schema `05-handover/REDESIGN_PROPOSAL_SCHEMA.md` (1.1.0) |
| **PATCH** | Corrections and clarifications with no structural impact | Fixed command, updated version number in `08-knowledge/android/version-matrix.md`, reworded ambiguous step, typo |

Rule of thumb: if a project onboarded yesterday would have to *rewrite already-consumed or
Accepted `docs/factory/` documents* to keep working with the new factory, the bump is MAJOR. If
it merely *gains options* — at most re-validating in-flight documents or reconciling section
names — MINOR. Otherwise, PATCH.

## Compatibility Policy

1. **Every target project records its factory version.** During P2 (Project Memory), the factory
   version from this file is written into the project's `docs/factory/PROJECT_STATE.md` (the
   state template provides the field). The `01-core/CLAUDE_MASTER.md` bootstrap (step 3a)
   compares that recorded version against this file at every session start; on a MAJOR mismatch
   it runs the upgrade procedure below before any phase work.
2. **MINOR and PATCH upgrades are free.** Projects onboarded under 1.x can use any later 1.y
   factory without document changes. Update the recorded version in `PROJECT_STATE.md`
   opportunistically at the next session.

   **1.0 → 1.1 note.** 1.1.0 is MINOR — additive plus reconciliation, no breaking contract
   changes — but projects onboarded under 1.0.0 should do two cheap reconciliation passes at
   their next session: (a) re-validate any **in-flight** (not yet Accepted) handovers against the
   extended schemas in `05-handover/` using `07-checklists/handover-validation.md`; (b) sweep
   `PROJECT_STATE.md` for pre-1.1 section names and rename them to the canonical set in
   `01-core/CONVENTIONS.md` (which lists the abolished names). Fully consumed handovers are
   historical records and stay as-is.
3. **MAJOR upgrade procedure.** When the factory MAJOR-bumps (e.g. 1.x → 2.0.0), each active
   project must be migrated before running further phases:
   1. Read the 2.0.0 entry in `10-releases/FRAMEWORK_CHANGELOG.md` — every MAJOR release entry
      lists exactly which contracts changed and the migration steps.
   2. **Re-validate live handover documents against the new schemas** in `05-handover/` using
      `07-checklists/handover-validation.md`. Handovers already fully consumed are historical
      records and are left as-is; handovers not yet consumed must be regenerated or amended as a
      new version (`<TYPE>_HANDOVER_v<N+1>.md`) conforming to the new schema.
   3. **Migrate the state file**: add/rename `PROJECT_STATE.md` sections to match the new
      `01-core/PROJECT_STATE_TEMPLATE.md`, preserving all content; do the same for
      `CHANGELOG.md` if its template changed.
   4. Update the recorded factory version in `PROJECT_STATE.md`, note the migration in the
      project's `CHANGELOG.md`, and commit before resuming any phase.
4. **Dormant projects** (no active phase) need no immediate migration; migrate at reactivation,
   before the first new session runs a prompt.

## Framework Release Process

1. Make the content changes on a branch (or directly, for PATCH) and decide the bump per the
   table above.
2. **Update this file and `10-releases/FRAMEWORK_CHANGELOG.md` together, in the same commit.**
   The changelog entry lists the changed files, the rationale, and — for MAJOR — the migration
   steps required of target projects. A version bump without a changelog entry, or vice versa, is
   an invalid release.
3. If the factory is git-managed, tag the release commit. Factory tags use the
   `factory/vX.Y.Z` format defined in `01-core/CONVENTIONS.md` (distinct from app release tags
   `release/vX.Y.Z` in target repos):

   ```powershell
   git tag factory/v1.1.0; git push origin factory/v1.1.0
   ```

   ```bash
   git tag factory/v1.1.0 && git push origin factory/v1.1.0
   ```

   Tags let a target project pin or reproduce the exact factory it was onboarded with.
4. There is no separate publish step — pulling the factory repo *is* the upgrade. Announce MAJOR
   releases to yourself by reviewing the changelog migration notes before the next project
   session.

## Deprecation Policy

1. A doc, prompt, or schema section slated for removal is **deprecated first, removed later** —
   never removed in the same release it is deprecated in.
2. Deprecated files get a banner immediately below the title, naming the replacement and the
   removal version:

   ```markdown
   > **DEPRECATED since 1.3.0 — will be removed in 1.4.0 (or the next MAJOR, whichever is
   > sooner). Use `04-aso/workflows/keyword-research.md` instead.**
   ```

3. The grace period is **at least one MINOR version**: deprecated in 1.3.0 means removable no
   earlier than 1.4.0. Removal of anything that target documents structurally depend on is a
   MAJOR bump regardless of grace period.
4. Deprecations and removals are always listed in their release's `FRAMEWORK_CHANGELOG.md` entry,
   and agents encountering a deprecation banner must follow the replacement path, not the
   deprecated one.
