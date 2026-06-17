# CONVENTIONS — Factory-Wide Contracts (Single Source of Truth)

This file is the authoritative registry of every cross-cutting convention in the
android-product-factory: IDs, scales, names, paths, git formats, the artifact-contract matrix,
template mechanics, and ownership pointers. Every other factory document cites this file instead
of restating these definitions. If any document restates a convention and disagrees with this
file, **this file wins** — record the conflict in the target app's `docs/factory/CHANGELOG.md`
so the factory can be fixed. Operating rules (roles, session procedure, escalation) belong to
`01-core/CLAUDE_MASTER.md`; phase definitions and gates belong to `01-core/PROJECT_LIFECYCLE.md`.

---

## 1. ID Registry

All identifiers use the format `PREFIX-NNN` with `NNN` zero-padded to 3 digits (`SCR-001`,
`IMPL-042`), except WS, which uses letters (`WS-A`, `WS-B`, …).

| Prefix | Meaning | Format | Owning artifact | Defined in prompt |
|---|---|---|---|---|
| SCR | Screen (inventory entry) | `SCR-001` | `audits/SCREEN_INVENTORY.md` | `01-core/prompts/05-design-audit.md` |
| DES | Design finding (per-screen) | `DES-001` | `audits/DESIGN_AUDIT.md` | `01-core/prompts/05-design-audit.md` |
| SYS | Systemic design finding (cross-screen) | `SYS-001` | `audits/DESIGN_AUDIT.md` | `01-core/prompts/05-design-audit.md` |
| A11Y | Accessibility finding (own stream) | `A11Y-001` | `audits/DESIGN_AUDIT.md` (accessibility section) | `01-core/prompts/05-design-audit.md` via `03-design/09-accessibility-review.md` |
| ENG | Engineering finding | `ENG-001` | `audits/ENGINEERING_AUDIT.md` | `01-core/prompts/03-modernization-audit.md` |
| ASO | ASO finding | `ASO-001` | `audits/ASO_AUDIT.md` | `01-core/prompts/12-aso-audit.md` |
| IMPL | Implementation ticket | `IMPL-001` | `handovers/IMPLEMENTATION_HANDOVER_v<N>.md` | `01-core/prompts/08-implementation-handover.md` |
| COPY | Copy-change ticket | `COPY-001` | `handovers/IMPLEMENTATION_HANDOVER_v<N>.md` | `01-core/prompts/08-implementation-handover.md` |
| ASSET | Asset deliverable | `ASSET-001` | `handovers/DESIGN_HANDOVER_v<N>.md` §Asset Manifest | `01-core/prompts/06-design-handover.md` |
| WS | Roadmap workstream | `WS-A` | `reports/MODERNIZATION_ROADMAP.md` | `01-core/prompts/04-roadmap-generation.md` |
| RISK | Risk register entry | `RISK-001` | `reports/RISK_REGISTER.md` | `01-core/prompts/03-modernization-audit.md` |
| KW | Keyword cluster | `KW-001` | `reports/KEYWORD_MAP.md` | `01-core/prompts/13-keyword-research.md` |
| EXP | Experiment | `EXP-001` | `handovers/ASO_HANDOVER_v<N>.md` EXP queue; executed in `reports/EXPERIMENT_LOG.md` | `01-core/prompts/14-store-assets.md` |

Rules:

- IDs are assigned once and never reused, renumbered, or recycled — including for rejected or
  deferred items. Gaps in the sequence are normal and MUST NOT be compacted.
- Cross-document references always use the full ID (`IMPL-004`), never bare numbers.
- The `A11Y-DES-xxx` composite format is **abolished**. Accessibility findings are their own
  stream: `A11Y-001`.
- Owning-artifact paths above are relative to `docs/factory/` in the TARGET repo (Section 5).

## 2. Scales & Verdicts

| Scale | Values (exact, no others valid) |
|---|---|
| Severity | Critical / High / Medium / Low |
| Effort | S / M / L / XL |
| Status | Not Started / In Progress / Blocked / Done / Deferred |
| Risk Likelihood | H / M / L |
| Risk Impact | H / M / L |
| Verification & handover verdicts | PASS / PASS with waivers / FAIL |

- The verdict word "CONDITIONAL" is **abolished**. A verdict with caveats is
  `PASS with waivers`, and every waiver MUST be itemized next to the verdict.
- Handover status lifecycle: **Draft → Delivered → Accepted | Rejected**. There is no "Partial"
  status.
- **Handover immutability rule:** a handover is immutable from the moment its status becomes
  **Accepted** (not "after seen"). After delivery, the only sections that may be appended to are
  `## Rejection Log` and `## Validation Appendix`. Any other change requires a new version
  (`_v<N+1>`, Section 7); the superseded version is retained.

## 3. Git Conventions

| Item | Format | Example |
|---|---|---|
| Work branch | `factory/p<N>-<slug>` | `factory/p6-redesign`, `factory/p4-agp-upgrade` |
| Commit message | `<type>(<scope>): P<N> <summary>` | `feat(ui): P6 IMPL-004 settings screen redesign` |
| Factory release tag | `factory/vX.Y.Z` | `factory/v1.1.0` |
| App release tag | `release/vX.Y.Z` | `release/v2.0.0` |

- Commit `<type>` uses the conventional set: `feat`, `fix`, `refactor`, `build`, `chore`,
  `docs`, `test`. `<scope>` is a module or concern (`ui`, `gradle`, `app`, `aso`).
- Reference ticket IDs in the summary when the commit implements one (`P6 IMPL-004 …`).
- **Abolished:** the `redesign/p6-ui` branch style and the `[P4]`-prefix commit-tag style.
  Sweep on sight.

## 4. Artifact-Contract Matrix (canonical copy)

Each prompt's output contract: artifact path (relative to `docs/factory/` in the TARGET repo),
the template it instantiates (`09-templates/` unless noted), the schema it must validate against
(`05-handover/`), and the gate checklist (`07-checklists/`). `—` = none.

| Prompt | Artifact path | Template | Schema | Gate checklist |
|---|---|---|---|---|
| 01 | `audits/DISCOVERY.md` — the ONLY artifact of P1; P1 does NOT create PROJECT_STATE.md | — | — | — |
| 02 | `PROJECT_STATE.md`, `CHANGELOG.md` | `01-core/PROJECT_STATE_TEMPLATE.md`, `01-core/CHANGELOG_TEMPLATE.md` | — | — |
| 03 | `audits/ENGINEERING_AUDIT.md`; `audits/deps-<config>.txt`; **creates** `reports/RISK_REGISTER.md` | `engineering-audit-report.md`; `risk-register.md` | — | — |
| 04 | `reports/MODERNIZATION_ROADMAP.md`; updates Backlog + risk register | — | — | — |
| P4 work | code (committed per step) | — | — | `engineering-modernization.md` |
| 05 | `audits/SCREEN_INVENTORY.md` (separate file, NOT embedded); `audits/DESIGN_AUDIT.md`; `assets/screenshots/baseline/` | `screen-inventory.md`; `design-audit-report.md` | — | — |
| 06 | `handovers/DESIGN_HANDOVER_v<N>.md` | — | `DESIGN_HANDOVER_SCHEMA.md` | `handover-validation.md` |
| 07 | `handovers/REDESIGN_PROPOSAL_v<N>.md` | — | `REDESIGN_PROPOSAL_SCHEMA.md` | `handover-validation.md` |
| 08 | `handovers/IMPLEMENTATION_HANDOVER_v<N>.md` | — | `IMPLEMENTATION_HANDOVER_SCHEMA.md` | `handover-validation.md` |
| 09 | code; `assets/screenshots/redesigned/`; ticket status table | — | — | `design-modernization.md` + `accessibility.md` |
| 10 | icon assets; `assets/store/<store>/` icon exports; `reports/ICON_SPEC.md` | — | — | — |
| 11 | `assets/store/<store>/<locale>/` (ALL uploadables incl. screenshots + feature graphic); `reports/SCREENSHOT_MANIFEST.md` | — | — | — |
| 12 | `audits/ASO_AUDIT.md` | `aso-audit-report.md` | — | — |
| 13 | `reports/KEYWORD_MAP.md` | `keyword-map.md` | — | — |
| 14 | `reports/STORE_LISTING_<store>_<locale>.md`; `handovers/ASO_HANDOVER_v<N>.md` | `store-listing.md` | `ASO_HANDOVER_SCHEMA.md` | `aso-launch.md` + `handover-validation.md` |
| 15 | `reports/VERIFICATION_REPORT_v<versionName>.md` (re-runs append `_r2`, `_r3`) | `verification-report.md` | — | `release-readiness.md` + `accessibility.md` |
| 16 | `handovers/RELEASE_HANDOVER_v<N>.md`; `reports/RELEASE_REPORT_v<versionName>.md` | `release-report.md` | `RELEASE_HANDOVER_SCHEMA.md` | `handover-validation.md` |
| post-release | `reports/EXPERIMENT_LOG.md`; KPI snapshots in PROJECT_STATE.md | `experiment-log.md` | — | `aso-post-release.md` |

`docs/factory/audits/ASO_AUDIT.md` is the only valid ASO_AUDIT path; any
`reports/ASO_AUDIT.md` reference is wrong. Experiment results are logged in
`reports/EXPERIMENT_LOG.md`, never in ASO_AUDIT.md.

## 5. Canonical Directory Tree (`docs/factory/` in the TARGET repo)

```
docs/factory/
├── PROJECT_STATE.md
├── CHANGELOG.md
├── audits/
│   ├── DISCOVERY.md
│   ├── ENGINEERING_AUDIT.md
│   ├── deps-<config>.txt
│   ├── SCREEN_INVENTORY.md
│   ├── DESIGN_AUDIT.md
│   └── ASO_AUDIT.md
├── handovers/
│   ├── DESIGN_HANDOVER_v<N>.md
│   ├── REDESIGN_PROPOSAL_v<N>.md
│   ├── IMPLEMENTATION_HANDOVER_v<N>.md
│   ├── ASO_HANDOVER_v<N>.md
│   └── RELEASE_HANDOVER_v<N>.md
├── reports/
│   ├── RISK_REGISTER.md
│   ├── MODERNIZATION_ROADMAP.md
│   ├── MODERNIZATION_REPORT.md          (P4; see PROJECT_LIFECYCLE.md)
│   ├── ICON_SPEC.md
│   ├── SCREENSHOT_MANIFEST.md
│   ├── KEYWORD_MAP.md
│   ├── STORE_LISTING_<store>_<locale>.md
│   ├── VERIFICATION_REPORT_v<versionName>.md
│   ├── RELEASE_REPORT_v<versionName>.md
│   └── EXPERIMENT_LOG.md
└── assets/
    ├── screenshots/
    │   ├── baseline/        (P5 working captures)
    │   └── redesigned/      (P6 working captures — "redesigned", never "redesign")
    └── store/<store>/<locale>/   (EVERYTHING uploadable: screenshots, feature graphic, icon exports)
```

Asset tree rules:

- `assets/screenshots/baseline/` and `assets/screenshots/redesigned/` hold **working captures**,
  named per Section 8. The directory is `redesigned`, never `redesign`.
- `assets/store/<store>/<locale>/` holds **every uploadable file** for that store and locale —
  store screenshots, feature graphic, icon exports, promo art. Nothing uploadable lives anywhere
  else. `<store>` ∈ `google-play`, `huawei-appgallery`, `rustore`; `<locale>` is a BCP-47 tag
  (`en-US`, `ru-RU`).
- **Abolished paths** (sweep on sight): `assets/screenshots/store/`, `assets/feature-graphic/`,
  `assets/icons/`, `reports/listings/`.

## 6. PROJECT_STATE.md Canonical Sections

Instantiated from `01-core/PROJECT_STATE_TEMPLATE.md`. Consumers MUST use exactly these section
names, in this order:

Header block · Current Status · Phase Tracker · App Identity · Tech Stack Snapshot (includes a
CI row) · Architecture Summary · Module Map · Store Presence · Known Risks · Open Questions ·
Decision Log · Backlog · Artifact Index (table: artifact, path, version, date, status) ·
Release History (versionName/code, date, rollout outcome) · Session Log.

- Canonical token for the application ID: `{{PACKAGE_ID}}`.
- **Abolished section names** (sweep from all consumers; never write, never cite): "Project
  Overview", "Tech Snapshot", "Risks & Open Questions", "Artifacts table", "Toolchain Versions",
  "Open Items", "Known Issues", "Distribution", "Release history" (lowercase variant).
- P2 status rule: the Phase Tracker shows P2 = In Progress until prompt 02's final
  self-verification passes, then Done.

## 7. File Versioning Semantics

| Artifact class | Suffix | Semantics | Example |
|---|---|---|---|
| Handovers & proposals | `_v<N>` | **Revision count**, starting at `_v1`. Every post-Accepted change (Section 2) bumps N. Superseded versions are retained, never deleted. | `DESIGN_HANDOVER_v2.md` |
| Verification reports | `_v<versionName>` | The **app versionName** under verification. Re-runs against the same versionName append `_r2`, `_r3`, … | `VERIFICATION_REPORT_v2.0.0.md`, `VERIFICATION_REPORT_v2.0.0_r2.md` |
| Release reports | `_v<versionName>` | The released app versionName; one report per release. | `RELEASE_REPORT_v2.0.0.md` |

Prompts reference handover versions generically as `_v<N>` — hardcoded `_v1` references are
abolished. The latest version is determined by the Artifact Index in PROJECT_STATE.md, never by
directory listing alone.

## 8. Screenshots, States & Demo Mode

- Screenshot file name: **`SCR-<id>-<state>-<theme>.png`** — e.g. `SCR-003-empty-dark.png`.
  `<id>` is the zero-padded screen number from SCREEN_INVENTORY (`003`, not `SCR-003-…-SCR-003`);
  `<state>` per the states contract below; `<theme>` ∈ `light` | `dark`.
- **States contract per screen:** `default` / `empty` / `loading` / `error` / `offline` — plus
  dark theme. Dark is captured **ALWAYS** for every inventoried screen (minimum:
  `default-dark`). A state that cannot be reproduced is recorded as such in the inventory's
  `States present` column, never silently skipped.
- **Demo clock: 10:00** in every capture, everywhere (status bar, in-app clocks). Demo-mode
  setup beyond the clock (full battery, full signal, no notification icons) follows
  `03-design/03-screenshot-capture.md`.

## 9. P1 Build Policy & Subagent Cap

- P1 (Discovery) permits **read-only Gradle only**: `./gradlew --version`, `./gradlew projects`,
  `./gradlew :app:dependencies`, plus **ONE optional `assembleDebug` attempt** whose failure is
  documented, not fixed. No other Gradle tasks, no code or build-file changes in P1.
- P1 gate wording: "builds OR build failure documented".
- **Subagent fan-out cap: 8 concurrent**, in every prompt and every phase. No exceptions.

## 10. Verification Tiers

Verification Tiers 1–3 are defined in and owned by **`02-engineering/07-build-verification.md`**.
There are exactly three tiers; any Tier 4/5 reference is legacy and maps into Tier 3. Cite that
file; never restate tier contents.

## 11. Template Instantiation Doctrine (one doctrine, both styles)

Two marker styles exist; the doctrine is identical for both:

- **01-core style** (`PROJECT_STATE_TEMPLATE.md`, `CHANGELOG_TEMPLATE.md`): the instantiable
  body sits between `=== TEMPLATE START ===` and `=== TEMPLATE END ===` markers.
- **09-templates style** (all files in `09-templates/`): the instantiable body sits below the
  `TEMPLATE BODY` marker comment, preceded by Usage and a Field Guide.

Mandatory steps, every instantiation:

1. **Copy** the template body (between/below the markers — markers, Usage, and Field Guide are
   never copied) into the instance path given by the template's Usage section / Section 4 above.
   Create parent directories if missing.
2. **Fill every `{{UPPER_SNAKE_CASE}}` token** with a real value. Unknown value → write `None`,
   `N/A`, or an explicit finding; never leave a token.
3. **Delete every guidance comment** (`<!-- … -->`) and **every example row** (marked
   `(example — replace)`).
4. **Verify zero tokens remain** — grep for `{{` over `docs/factory/` and require empty output:

   ```bash
   grep -rn "{{" docs/factory/ && echo "FAIL: unfilled tokens" || echo "PASS"
   ```

   ```powershell
   Select-String -Pattern '\{\{' -Path docs/factory/**/*.md
   # Empty output = PASS; any hit = FAIL.
   ```

5. **Record the instantiation** in `docs/factory/CHANGELOG.md` per the prompt's Documentation
   Update Rules.

An instance containing `{{`, a guidance comment, or an example row fails
`07-checklists/handover-validation.md` and MUST NOT be consumed downstream. Templates are the
only factory files allowed to contain `{{…}}` tokens; the factory copy is never edited during
project work.

## 12. Ownership Pointers (numbers live there, not here)

| Convention | Sole owner | Everyone else |
|---|---|---|
| Keyword density (2–4 natural uses of a primary keyword per 4000-char description, hard cap 5) | `08-knowledge/aso/ranking-factors.md` | Cite, never restate |
| Keyword seed/candidate volumes (20–40 seeds / 100–200 candidates) | `04-aso/workflows/keyword-research.md` | Cite (prompt 13 cites) |
| Rollout stages, dwell times, HOLD thresholds (10→25→50→100%, ≥24h dwell, crash +0.2pp / ANR ≥0.40%) | `10-releases/release-workflow.md` **stage 4** | Cite; prompt 16 executes stages 4–5 by reference |
| Play Vitals bad-behavior thresholds & startup-time bands | `08-knowledge/android/play-vitals-performance.md` | Cite |
| Design audit rubric (H1–H9, scored per screen) | `03-design/01-design-audit.md` | Cite; the report template carries the scorecard |
| Icon redesign procedure (mono stroke ~2dp; brief in `reports/ICON_SPEC.md`) | `03-design/06-icon-redesign.md` | Prompt 10 defers to it |
| Splash redesign | `03-design/07-splash-redesign.md`, executed as a prompt 09 ticket | Cite |
| Motion vocabulary (duration scale, easing, transition decision table, reduced-motion rule) | `08-knowledge/design/material3-essentials.md` + method `03-design/10-motion-design.md` | Cite |
| Merged-manifest path (both AGP variants) | `02-engineering/06-manifest-audit.md` Step 0 | Cite |
| Verification tiers 1–3 | `02-engineering/07-build-verification.md` | Cite |
| Risk register creation | Prompt 03 (P3) instantiates `09-templates/risk-register.md`; prompt 04 adds workstream risks; reviewed at every phase gate | Never claim prompt 02 or 04 creates it |

## 13. String Freeze & COPY Tickets

- Prompt 06 declares frozen vs redesignable strings (DESIGN_HANDOVER §Localization & Content
  Constraints).
- Prompt 07 may propose copy changes; prompt 08 emits them as `COPY-NNN` tickets; prompt 09
  implements COPY tickets.
- There is no blanket string freeze — but **un-ticketed string changes remain banned**.

## 14. Writing Rules (factory documents)

1. Every command block is labeled `bash` or `powershell`; non-gradlew shell commands get both
   variants where they differ (`mkdir -p` ↔ `New-Item -ItemType Directory -Force`).
2. No placeholder text outside templates (Section 11).
3. Conventions are defined here once; everywhere else: one line + a pointer to this file.
4. The handover-validation header check means "all header fields the handover type's schema
   requires" — never an invented field list. Its ID grep covers the full Section 1 registry.
