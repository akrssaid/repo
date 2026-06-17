# QUICK_REFERENCE — Operator Card

One page for driving the factory. Authority lives elsewhere: rules in `01-core/CLAUDE_MASTER.md`,
phases in `01-core/PROJECT_LIFECYCLE.md`, naming/paths/IDs in `01-core/CONVENTIONS.md`.

## Phase table

| Phase | Run this prompt | Artifact produced (target `docs/factory/`) | Gate to pass | Human decision |
|---|---|---|---|---|
| P1 Discovery | `01-core/prompts/01-discovery.md` | `audits/DISCOVERY.md` | Builds OR build failure documented; every claim cited | Confirm target repo + engagement goal |
| P2 Project Memory | `01-core/prompts/02-project-memory.md` | `PROJECT_STATE.md`, `CHANGELOG.md` | Zero `{{` tokens; facts traceable to DISCOVERY.md | Answer open questions (identity, stores) |
| P3 Audit & Roadmap | `03-modernization-audit.md`, then `04-roadmap-generation.md` | `audits/ENGINEERING_AUDIT.md`, `reports/RISK_REGISTER.md`, `reports/MODERNIZATION_ROADMAP.md` | Findings have severity/effort/evidence; ordered roadmap | **Accept the roadmap** (recorded in Decision Log) |
| P4 Modernization | `02-engineering/migrations/*` guides | migrated code, `reports/MODERNIZATION_REPORT.md` | `07-checklists/engineering-modernization.md` + build verification | Approve any scope change to the roadmap |
| P5 Design Audit & Handover | `05-design-audit.md` → `06-design-handover.md` → `07-design-redesign.md` (into Claude Design) → `08-implementation-handover.md` | `audits/SCREEN_INVENTORY.md`, `audits/DESIGN_AUDIT.md`, `handovers/DESIGN_HANDOVER_v<N>.md`, `REDESIGN_PROPOSAL_v<N>.md`, `IMPLEMENTATION_HANDOVER_v<N>.md` | `07-checklists/handover-validation.md` per handover | **Accept or reject each handover** |
| P6 Redesign Implementation | `09-ui-implementation.md`, `10-icon-redesign.md`, `11-screenshot-generation.md` | UI code, icon + `reports/ICON_SPEC.md`, `assets/store/<store>/<locale>/`, `reports/SCREENSHOT_MANIFEST.md` | `design-modernization.md` + `accessibility.md` + build verification | Approve visual direction deviations |
| P7 ASO & Store Assets | `12-aso-audit.md`, `13-keyword-research.md`, `14-store-assets.md` | `audits/ASO_AUDIT.md`, `reports/KEYWORD_MAP.md`, `reports/STORE_LISTING_<store>_<locale>.md`, `handovers/ASO_HANDOVER_v<N>.md` | `aso-launch.md` + `handover-validation.md` | **Accept final copy** (it's FINAL after acceptance) |
| P8 Verification & Release | `15-final-verification.md`, then `16-release-preparation.md` | `reports/VERIFICATION_REPORT_v<versionName>.md`, `handovers/RELEASE_HANDOVER_v<N>.md`, `reports/RELEASE_REPORT_v<versionName>.md` | `release-readiness.md`; verdict PASS or PASS with waivers | **Sign the build, press publish, advance each rollout stage** |

## First message to paste into Claude Code

**(a) Brand-new project**

```
Load <factory-path>/01-core/CLAUDE_MASTER.md in full and follow it.
Target repo: <absolute path to the Android app>.
App: <name>. Engagement goal: <full lifecycle | specific outcome>.
No docs/factory/ exists yet — start with P1 Discovery
(01-core/prompts/01-discovery.md).
```

**(b) Resuming mid-lifecycle**

```
Load <factory-path>/01-core/CLAUDE_MASTER.md in full and follow it.
Target repo: <absolute path>. Run the Session Bootstrap Procedure:
read docs/factory/PROJECT_STATE.md and the CHANGELOG tail, announce
the current phase, and continue from the recorded next action.
```

**(c) ASO-only engagement**

```
Load <factory-path>/01-core/CLAUDE_MASTER.md in full and follow it.
Target repo: <absolute path>. Engagement: ASO-only (modular entry at P7
per 01-core/PROJECT_LIFECYCLE.md). P1+P2 are still mandatory — run them
compressed if docs/factory/ is missing, then proceed to prompt 12.
Stores in scope: <google-play | huawei-appgallery | rustore>.
```

## Five most-used checklists (`07-checklists/`)

| Checklist | Used when |
|---|---|
| `handover-validation.md` | Every handover/proposal delivery (P5, P7, P8) — validates against its `05-handover/` schema |
| `engineering-modernization.md` | Closing P4 |
| `design-modernization.md` (+ `accessibility.md`) | Closing P6 |
| `aso-launch.md` | Closing P7 |
| `release-readiness.md` | Closing P8 |

## Naming cheat sheet (full definitions: `01-core/CONVENTIONS.md`)

| Thing | Format | Example |
|---|---|---|
| Work branch | `factory/p<N>-<slug>` | `factory/p6-redesign` |
| Commit | `<type>(<scope>): P<N> <summary>` | `feat(ui): P6 IMPL-004 settings redesign` |
| Finding / ticket IDs | `PREFIX-NNN` | `ENG-003`, `DES-004`, `IMPL-006`, `COPY-002` |
| Screenshot | `SCR-<id>-<state>-<theme>.png` | `SCR-001-empty-dark.png` |
| Handover / proposal | `_v<N>` (revision count) | `DESIGN_HANDOVER_v2.md` |
| Verification / release report | `_v<versionName>` (re-runs `_r2`) | `VERIFICATION_REPORT_v2.0.0_r2.md` |
| Tags | `factory/vX.Y.Z` (factory), `release/vX.Y.Z` (app) | `release/v2.0.0` |
| Verdicts | PASS / PASS with waivers / FAIL | — |

## Loop-backs at a glance

| Failure | Goes back to | Then |
|---|---|---|
| Audit reveals discovery gap (P3/P5) | P1 — amend DISCOVERY.md | Return to the phase that found it |
| Migration step blocked (P4) | P3 — re-plan remaining roadmap | Completed steps stand |
| Defective handover found in P6 | P5 — issue `_v<N+1>` (old version retained) | Resume P6 against the new version |
| P8 verification failure | P4 (code) / P6 (UI) / P7 (asset or listing) | Fast-forward straight back to P8; re-run only the failed check |

Loop-backs are scoped repairs, never restarts (`01-core/PROJECT_LIFECYCLE.md`).

## Where everything lives

- **Factory repo (this repo): methods, never project data.** Prompts `01-core/prompts/`,
  procedures `02-engineering/`–`04-aso/`, schemas `05-handover/`, playbooks `06-playbooks/`,
  checklists `07-checklists/`, knowledge `08-knowledge/`, templates `09-templates/`, release
  policy `10-releases/`. Read-only during project work.
- **Target repo `docs/factory/`: ALL per-project artifacts.** `PROJECT_STATE.md` (current
  truth), `CHANGELOG.md` (history), `audits/`, `handovers/`, `reports/`, `assets/`. Canonical
  tree and naming: `01-core/CONVENTIONS.md` Section 5.

## Escalation triggers — Claude stops and asks you when…

1. **Scope is ambiguous** — multiple plausible interpretations, missing prerequisites, or work
   spanning more than one phase (CLAUDE_MASTER bootstrap step 8).
2. **A gate decision is yours** — roadmap acceptance (P3), handover acceptance/rejection
   (P5/P7/P8), final store copy, "publish", every rollout-stage advance.
3. **Signing or credentials are needed** — keystores, store-console access, API keys. Claude
   never handles or asks you to paste secrets into chat.
4. **A gate fails and the fix is out of scope** — loop-back target proposed, your call to
   proceed.
5. **Destructive or irreversible actions** — force-push, history rewrite, deleting artifacts,
   anything affecting production users.
6. **Factory-version drift** — PROJECT_STATE.md records a different MAJOR factory version than
   `VERSION.md`; the upgrade procedure runs before any phase work.
7. **The target doesn't match assumptions** — not a Gradle Android project, target repo
   ambiguous, or store presence can't be verified.
