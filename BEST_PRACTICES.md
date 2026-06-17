# BEST PRACTICES

Operating discipline for running the Android Product Factory: how to structure AI sessions, keep
documentation trustworthy, commit safely, treat handovers as contracts, and manage risk. These
practices are binding for Claude agents executing factory prompts and strongly recommended for the
human operator; where a prompt file and this document conflict, the prompt file wins for its phase.

## AI Workflow Best Practices

1. **One lifecycle phase per session.** A session that starts in P3 ends in P3. Phases have
   different personas, inputs, and acceptance criteria; mixing them produces artifacts that pass
   no gate cleanly. If a phase is too large for one session (P4 migrations, P6 screens), split it
   into multiple sessions *within* the phase — never span two phases.
2. **Fresh context per phase.** Start every phase with a new Claude Code session. Stale context
   carries assumptions that were corrected at the previous gate; the corrected truth lives in
   `docs/factory/PROJECT_STATE.md`, not in the old conversation.
3. **State docs are the only cross-session memory.** Anything an agent learned that future
   sessions need must be written into `PROJECT_STATE.md`, the audits, or a handover before the
   session ends. If it is not in `docs/factory/`, it did not happen. Conversely, every session
   begins by reading `PROJECT_STATE.md` — never by asking the human to re-explain the project.
4. **Verification gates before phase transitions.** A phase is Done only when its prompt's
   Acceptance Criteria pass and the matching checklist in `07-checklists/` is checked off. The
   human reviews the gate artifact (audit, roadmap, handover, report) before the next phase
   starts. No gate, no transition — even when the output "looks obviously fine".
5. **Fan out subagents for breadth, stay single-threaded for depth.** Fan out when work is
   read-only and parallelizable: dependency analysis across modules, screen inventory, keyword
   research across locales, competitor analysis. Each subagent gets a narrow question and returns
   a structured summary the lead agent merges. Stay single-threaded whenever work mutates the
   repo (migrations, UI implementation), depends on sequential decisions, or writes to
   `PROJECT_STATE.md` — exactly one writer per state file per session. Each prompt's
   `## Agent Orchestration` section gives the phase-specific rule.
6. **Never let an agent both produce and accept its own handover.** The producer is biased toward
   its own output. Validation against `07-checklists/handover-validation.md` happens in a separate
   fresh session (or by the human). The validator's job is to reject: missing schema sections,
   unverifiable claims, and ambiguity a zero-context implementer could misread are all grounds for
   bouncing the handover back.

## Documentation Best Practices

1. **Update `PROJECT_STATE.md` at the end of every session** — not "when something big happens".
   Minimum update: current phase and status, what was completed this session, what is next, any
   new blockers, any new decisions in the decision log. A session that ends without a state update
   is an unfinished session.
2. **Changelog discipline.** Every user-visible or build-visible change gets a line in
   `docs/factory/CHANGELOG.md` in the same session it was made, under the unreleased heading, in
   the format defined by `01-core/CHANGELOG_TEMPLATE.md`. Release notes at P8 are *assembled* from
   the changelog, never reconstructed from memory or git archaeology.
3. **Use the decision log for decisions, not narration.** Record: what was decided, the
   alternatives rejected, why, who decided (human vs. agent-proposed/human-ratified), and the
   date. Future sessions consult the decision log before re-opening any settled question; a
   settled decision is only re-opened by the human, explicitly.
4. **Project docs live in the target repo, never in the factory.** The factory is reusable
   instructions; it must contain zero app-specific facts. If you find yourself wanting to write
   "for this app, the minSdk is..." into a factory file, you are in the wrong repo — it belongs in
   the target app's `docs/factory/PROJECT_STATE.md`. The reverse also holds: generic lessons
   learned that would help every project are factory improvements — propose them as a factory
   change with a `FRAMEWORK_CHANGELOG.md` entry, separately from project work.
5. **Conventions discipline.** Never restate a convention — ID formats, scales, naming, paths,
   git formats are defined once in `01-core/CONVENTIONS.md`; everywhere else, cite it. A restated
   convention is a future contradiction: when the canonical definition moves, the copy silently
   diverges and zero-context agents follow whichever they read first. If a document needs a
   convention, it writes one line plus the pointer.

## Commit Strategy

1. **One commit format, defined in `01-core/CONVENTIONS.md`:** `<type>(<scope>): P<N> <summary>`.
   The phase lives in the subject, always in that position — there is no alternative tag style.
   Reference roadmap items, ticket IDs, and verification evidence in the body:

   ```
   build(gradle): P4 migrate to version catalogs

   Per docs/factory/reports/MODERNIZATION_ROADMAP.md item 3. Verified with
   ./gradlew build and 02-engineering/07-build-verification.md.
   ```

2. **Per-phase branches.** Mutating phases work on a branch named `factory/p<N>-<slug>` per
   `01-core/CONVENTIONS.md`, e.g. `factory/p4-modernization`, `factory/p6-redesign`. Merge to the
   mainline only after the phase gate passes. Documentation-only phases (P1–P3, P5, P7 audits)
   may commit `docs/factory/` directly to mainline.
3. **Atomic commits.** One logical change per commit: one migration step, one screen, one listing
   locale. Every commit must build (`./gradlew assembleDebug` at minimum). Atomicity is what makes
   rollback plans executable — `git revert` of a single commit must be a meaningful undo.
4. **Never mix refactor and feature** (or refactor and migration) in one commit. A diff that
   changes behavior and shape simultaneously cannot be reviewed or reverted safely. Sequence:
   refactor commit (no behavior change, tests green) → feature/migration commit.
5. **What never gets committed:** keystores and signing configs with secrets (`*.jks`,
   `*.keystore`, `keystore.properties`), API keys, `local.properties`, `google-services.json`
   variants containing private keys, Play Console service-account JSON, build outputs
   (`build/`, `*.apk`, `*.aab` — release artifacts go to the console, not git), and generated
   screenshots larger than the store needs. Keep these in `.gitignore` and verify with
   `git status` before every commit; an agent that notices a secret already tracked must stop and
   escalate to the human rather than commit on top of it.

## Handover Strategy

1. **Schemas are contracts.** Five governed document types live in `docs/factory/handovers/`:
   the DESIGN, IMPLEMENTATION, ASO, and RELEASE handovers (`<TYPE>_HANDOVER_v<N>.md`) and the
   `REDESIGN_PROPOSAL_v<N>.md` returned by Claude Design. Each must conform exactly to its schema
   in `05-handover/`. The schema defines required sections, field semantics, and completeness
   rules. A consumer (Claude Design, an implementing agent, the human uploading to a store) is
   entitled to assume the contract holds and must not have to ask clarifying questions.
2. **Validate before consumption, every time.** Run `07-checklists/handover-validation.md` against
   the handover in a fresh session before any consumer touches it (see AI Workflow rule 6).
   Validation result (pass, or rejection with reasons) is recorded in `PROJECT_STATE.md`.
3. **Handovers are versioned and immutable once Accepted.** The immutability rule is defined by
   the schemas in `05-handover/`: after a document's status reaches **Accepted**, its content is
   frozen; the only sections that may still be appended to are the **Rejection Log** and the
   **Validation Appendix**. Substantive changes produce `v<N+1>` with a short "Changes from
   v<N>" section at the top. This keeps the round trip with Claude Design auditable — you can
   always see exactly what each proposal was responding to.
4. **When a handover is rejected:** the validator writes the rejection reasons into the document's
   Rejection Log and `PROJECT_STATE.md`; the producing phase re-opens (status back to
   In Progress); a fresh session re-runs the producing prompt with the rejection reasons as
   additional input; the fixed handover ships as `v<N+1>`. Never "fix it inline" in the consumer's
   session — that destroys the contract boundary and hides the defect from the process.

## Risk Management

1. **Maintain a live risk register.** The register is created at P3 by
   `01-core/prompts/03-modernization-audit.md`, which instantiates `09-templates/risk-register.md`
   into `docs/factory/reports/RISK_REGISTER.md`; prompt 04 adds workstream risks, and every later
   phase updates it whenever it introduces or retires a risk. Each entry uses the risk scales from
   `01-core/CONVENTIONS.md`: `RISK-NNN` ID, description, **Likelihood (H/M/L)**, **Impact
   (H/M/L)**, mitigation, owner, status. The register is reviewed at every phase gate; the P8 gate
   reads it last — unmitigated high-likelihood/high-impact risks block release.
2. **Rollback plan before any risky migration.** Before SDK/AGP/Gradle/Kotlin migrations or
   storage-format changes, write down (in the risk register or roadmap item): the pre-migration
   tag to revert to, the exact revert command, the verification that confirms rollback worked, and
   any irreversible step (e.g. a Room schema migration shipped to users) — irreversible steps get
   extra gate scrutiny.
3. **Pre-migration git tags.** Tag before every migration so rollback is one command:

   ```powershell
   git tag pre-agp-8.9-migration; git push origin pre-agp-8.9-migration
   # rollback if needed:
   git revert <migration-commit-sha>   # preferred over reset on shared branches
   ```

   ```bash
   git tag pre-agp-8.9-migration && git push origin pre-agp-8.9-migration
   # rollback if needed:
   git revert <migration-commit-sha>   # preferred over reset on shared branches
   ```

4. **Staged rollouts, always.** Never release to 100% directly. Follow
   `10-releases/release-workflow.md`: start at ~10%, watch crash-free rate and ANR vitals, promote
   in steps, and halt/rollback per the criteria in the RELEASE handover. A halted rollout is a
   cheap incident; a 100% release of a broken build is not.
5. **Keystore and signing safety.** The upload/signing keystore is the single most irreplaceable
   asset of a published app — losing it (without Play App Signing) means losing the listing.
   Rules: keystore files and passwords never enter git or any handover document; keep at least two
   offline backups; agents never create, replace, or rotate keystores; enroll in Play App Signing
   so Google holds the app signing key and the upload key is recoverable. Signing steps in P8 are
   executed by the human or with the human watching.
