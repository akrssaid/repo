# 06-playbooks — Category Playbooks

This directory contains category-specific expertise that layers ON TOP of the generic lifecycle
defined in `01-core/PROJECT_LIFECYCLE.md`. The lifecycle tells the agent *how* to take any Android
app from discovery to release; a playbook tells the agent *what is true about this specific app
category* — its market shape, technical patterns, UX conventions, ASO realities, monetization
norms, and policy landmines. A playbook never replaces a lifecycle prompt; it sharpens every phase
with category knowledge.

## How playbooks plug into the lifecycle

1. **Load at P1 (Discovery).** While executing `01-core/prompts/01-discovery.md`, the agent
   classifies the target app. If the category matches a playbook in this directory, the agent
   reads the full playbook immediately and records the match in
   `docs/factory/PROJECT_STATE.md` under the project facts (e.g.
   `Playbook: 06-playbooks/file-manager.md`).
2. **Keep loaded for the whole project.** The playbook stays in context for every subsequent
   phase. Subagents spawned during any phase must be given the playbook path in their briefing.
3. **Referenced explicitly by these prompts:**

| Prompt | Phase | What it pulls from the playbook |
|---|---|---|
| `01-core/prompts/03-modernization-audit.md` | P3 | Engineering Patterns, Common Pitfalls (technical traps, policy-sensitive permissions/SDKs) |
| `01-core/prompts/05-design-audit.md` | P5 | Design Patterns (category UX conventions, key screens, what top apps do) |
| `01-core/prompts/12-aso-audit.md` | P7 | ASO Patterns, Monetization Patterns, Common Pitfalls (metadata landmines); category benchmark CVR/rating-norm |
| `01-core/prompts/13-keyword-research.md` | P7 | ASO Patterns (keyword themes, intent phrasings, title formulas); the per-category keyword **seed list** consumed by `04-aso/workflows/keyword-research.md` Stage 1 |

4. **Recommended Workflow section is binding guidance.** Each playbook ends with a
   `## Recommended Workflow` section that re-weights lifecycle phases for the category (e.g. for
   `video-downloader.md`, policy assessment dominates and happens before any engineering spend).
   When the playbook and the generic lifecycle conflict on emphasis or ordering *within* a phase,
   the playbook wins; phase boundaries and artifact contracts always follow
   `01-core/PROJECT_LIFECYCLE.md`.

## No playbook match

If the target app fits no playbook, proceed with the generic lifecycle plus
`08-knowledge/stores/policy-landmines.md` for policy awareness, and note
`Playbook: none` in `docs/factory/PROJECT_STATE.md`. Consider writing a new playbook
(see below) if the operator will publish more than one app in that category.

## Playbook section contract

Every playbook in this directory is 250–400 lines and contains these H2 sections, in this
order (H3 subsections within them are free-form):

| # | Section | Job |
|---|---|---|
| 1 | `## Category Snapshot` | Market reality, user intent, competition shape — grounds all later decisions |
| 2 | `## Engineering Patterns` | Core technical components, recommended libraries **with license notes**, architecture specifics, performance traps |
| 3 | `## Design Patterns` | UX conventions users expect, key screens and their jobs, what top apps do |
| 4 | `## ASO Patterns` | Keyword themes and intent phrasings, screenshot messaging that converts, title formulas |
| 5 | `## Monetization Patterns` | What monetizes here: ad placements that work without destroying UX, paywall placement, typical free/pro split |
| 6 | `## Common Pitfalls` | Policy landmines SPECIFIC to the category, technical traps, review-bomb causes |
| 7 | `## Recommended Workflow` | Which lifecycle phases need category-specific emphasis, in order |

Some categories carry an existential risk that must be stated before anything else (e.g.
policy in `video-downloader.md`, honesty in `data-recovery.md`); such playbooks open with a
short bolded warning block between the purpose paragraph and `## Category Snapshot`, and may
add one extra prominent H2 dedicated to that risk (e.g. the policy decision gate in
`video-downloader.md`, placed after `## Engineering Patterns`).

### Two data deliverables every playbook must carry

Beyond the seven sections, each playbook supplies two short, named data assets that downstream
prompts consume directly. Keep them current; both are best-effort estimates, never invented
precision:

1. **Category benchmark (est.).** A one- or two-line "what normal looks like" figure for the
   category — a typical store-listing **conversion rate (CVR)** band and a **rating norm** (the
   average star rating a credible app in this category sustains). Lives inside `## ASO Patterns`
   (CVR) and `## Monetization Patterns`/`## ASO Patterns` (rating norm), explicitly labelled
   `(est.)`. Prompt 12 reads it to size the gap between the target app and the category and to
   set realistic CVR/rating goals; the ASO_HANDOVER Baseline Metrics Snapshot (per `_V11_SPEC`
   D16) records the target's actuals against this benchmark.
2. **Keyword seed list.** A ready-to-use block of **20–40 category seeds** (head, task, and
   long-tail intent phrasings) inside `## ASO Patterns`. This is exactly the per-category seed
   source `04-aso/workflows/keyword-research.md` **Stage 1** pulls when a playbook matches; the
   agent treats every seed as a forced inclusion before its own expansion. Seed/candidate volumes
   themselves (20–40 → 100–200) are owned by `keyword-research.md`, not restated here.

## Writing a new playbook for a new category

1. Copy the section contract above verbatim — same seven H2 sections, same order, 250–400 lines.
2. Start with `# <Category> Playbook` and a one-paragraph purpose statement naming the category
   and the apps it covers.
3. Research before writing: top 10 Play Store apps in the category (features, reviews — especially
   1-star reviews, which reveal review-bomb causes), current Play policy pages touching the
   category, and the real library landscape including licenses (AGPL and GPL are traps for
   closed-source apps — always note license per library).
4. Lead with the existential risk if the category has one. If a category is policy-critical or
   honesty-critical, that fact shapes every other section and must appear first.
5. Make Engineering Patterns concrete: name libraries, APIs, and permissions; align with the tech
   baseline in `_BUILD_SPEC` conventions as mirrored in `08-knowledge/android/modern-stack.md`.
6. Make Common Pitfalls falsifiable: each pitfall should be checkable during P3/P7 audits.
7. Cross-reference factory files by repo-relative path; never duplicate generic lifecycle content.
8. Add the new file to this README's directory listing context and register the category trigger
   in your P1 classification notes.

## Files

| File | Category | Risk posture |
|---|---|---|
| `06-playbooks/document-reader.md` | PDF/Office/eBook readers | License traps (AGPL), all-files-access rejection |
| `06-playbooks/video-downloader.md` | Video/media downloaders | POLICY-CRITICAL — often not Play-viable |
| `06-playbooks/file-manager.md` | File managers / explorers | Storage-access declarations, data-loss reputation risk |
| `06-playbooks/data-recovery.md` | Photo/data recovery | HONESTY-CRITICAL — misleading-claims enforcement |
| `06-playbooks/pdf-scanner.md` | Document/PDF scanners (camera → PDF/OCR) | Trust collapse (CamScanner-style), watermark resentment, OCR overclaiming |
| `06-playbooks/qr-barcode-scanner.md` | QR/barcode scanners & generators | OS-commoditized; malicious-QR liability, interstitial review-bombs |
