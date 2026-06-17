# 08-knowledge — Knowledge Base Map

This directory is the factory's evergreen reference library: facts, doctrine, and decision tables
that any agent may consult during **any** lifecycle phase (P1–P8). Knowledge docs answer "what is
true and what do we believe" — they never contain step-by-step procedures. Procedures live in
`02-engineering/`, `03-design/`, and `04-aso/`; prompts live in `01-core/prompts/`. When a
procedure needs a fact (an API level, a policy rule, a color-role definition), it cross-references
a doc here instead of restating it.

Conventions (IDs, scales, naming, canonical paths, the keyword-density rule, the per-screen states
contract) are defined once in `01-core/CONVENTIONS.md`; knowledge docs cite it rather than
restating any convention.

**last reviewed: 2026-06-10**

---

## Categories

| Category | Directory | Scope |
|---|---|---|
| Android platform & toolchain | `08-knowledge/android/` | API levels, the reference stack, engineering pitfalls, **Play vitals & performance thresholds** |
| Design | `08-knowledge/design/` | Material 3 facts (incl. motion), UX doctrine, **UX-writing reference**, icon doctrine |
| ASO | `08-knowledge/aso/` | How store ranking and conversion actually work |
| Stores | `08-knowledge/stores/` | Policy hazards, cross-store differences, **per-SDK Data-safety mapping** |
| Monetization | `08-knowledge/monetization/` | Ads and subscription pattern doctrine |

---

## All 15 Knowledge Docs

| Doc | One-line summary |
|---|---|
| `08-knowledge/android/version-matrix.md` | API 21→36 behavior-change matrix, minSdk reach tradeoffs, target-API deadline mechanics, canonical AGP↔Gradle↔Kotlin↔JDK table. |
| `08-knowledge/android/modern-stack.md` | The opinionated 2026 reference stack — each choice, its rationale, and the legacy tech it replaces. |
| `08-knowledge/android/common-pitfalls.md` | Hard-won pitfall catalog (build, runtime, UI, release) with Symptom / Cause / Fix / Prevention for each. |
| `08-knowledge/android/play-vitals-performance.md` | Canonical numeric home for Play vitals bad-behavior thresholds, startup-time bands, frame/jank thresholds, and the rollout baselines release docs cite. |
| `08-knowledge/design/material3-essentials.md` | M3 working reference: color roles, type scale, shape, elevation, motion (duration/easing/transition patterns), component selection, M2→M3 diff. |
| `08-knowledge/design/mobile-ux-principles.md` | Evergreen UX doctrine for utility apps: core-task-first, state completeness, forgiveness, ad-UX coexistence. |
| `08-knowledge/design/ux-writing-essentials.md` | UX-writing reference: error/empty/permission-primer copy patterns, button-label rules, length budgets & localization expansion, the COPY-ticket flow. |
| `08-knowledge/design/app-icon-principles.md` | Icon doctrine: the icon's three jobs, composition rules, scalability test, adaptive-icon recap, worked examples. |
| `08-knowledge/aso/ranking-factors.md` | What actually moves store ranking — keyword placement weights, the canonical keyword-density rule, velocity, ratings, paid-UA × ASO interaction. |
| `08-knowledge/aso/conversion-optimization.md` | Listing-page conversion levers: icon, screenshots, title/short description, ratings display, A/B doctrine. |
| `08-knowledge/stores/policy-landmines.md` | Policy areas that get apps rejected or removed — permissions, data safety, ads, content, account deletion. |
| `08-knowledge/stores/store-comparison.md` | Google Play vs Huawei AppGallery vs RuStore: review process, asset specs, monetization, update mechanics. |
| `08-knowledge/stores/data-safety-mapping.md` | Per-SDK Data-safety answer reference: the question model + a common-SDK mapping table; pairs with the data-safety listing workflow. |
| `08-knowledge/monetization/ads-patterns.md` | Ad format selection, placement doctrine, eCPM realities, policy boundaries for ad-funded utility apps. |
| `08-knowledge/monetization/subscription-patterns.md` | Subscription design for utility apps: paywall placement, trial mechanics, pricing doctrine, churn realities. |

---

## Freshness Doctrine

Knowledge decays at different rates. The rules:

1. **Every doc carries a `last reviewed: YYYY-MM-DD` line near the top.** An agent citing a fact
   from a doc whose review date is more than ~12 months old must verify that fact against current
   official documentation before acting on it.
2. **Review on factory MINOR releases.** Whenever the factory version (see `VERSION.md` and
   `10-releases/FRAMEWORK_CHANGELOG.md`) increments its MINOR component, sweep all 15 docs:
   re-verify drift-prone facts, update the review date, log changes in the framework changelog.
3. **Store-policy material is re-verified before every release cycle.** Anything in
   `08-knowledge/stores/` (including `stores/data-safety-mapping.md`, whose per-SDK disclosures
   track SDK behavior and policy) — plus the target-API deadline section of
   `android/version-matrix.md`, the policy boundaries in `08-knowledge/monetization/ads-patterns.md`,
   and the Play vitals bad-behavior thresholds in `android/play-vitals-performance.md` (the numbers
   that gate visibility and the rollout baselines reference) — must be checked against the live
   policy/Console pages before executing P7/P8 on any target app. Policies and thresholds change
   without notice; a stale fact here can cost an app its listing.
4. **Drift-prone facts are flagged inline.** Docs mark volatile facts (versions, deadlines, fees,
   percentages) with a "verify against current official docs" note. Stable doctrine (e.g. "one
   metaphor per icon") carries no flag and needs no verification.

---

## Knowledge Docs vs Method Docs

| | Knowledge docs (`08-knowledge/`) | Method docs (`02-engineering/`, `03-design/`, `04-aso/`) |
|---|---|---|
| Contain | Facts, tables, doctrine, decision rules | Numbered procedures, commands, workflows |
| Answer | "What is true? What do we believe? Which option when?" | "How do I do this, step by step?" |
| Phase-bound | No — consulted from any phase | Yes — tied to specific lifecycle phases |
| Change cadence | On review sweeps (freshness doctrine above) | When the method itself improves |
| Example | API 33 requires notification permission | How to run the manifest audit that detects it |

Rule of thumb: if a sentence starts with an imperative verb and belongs in a sequence, it is
method, not knowledge — move it to the right `02`/`03`/`04` doc and cross-reference it from here.

---

## Usage Notes for Agents

- Cite knowledge docs by repo-relative path in audits and handovers so reviewers can trace claims.
- Never copy a volatile fact into a per-project artifact without its date context (e.g. write
  "target-API floor as of 2026-06" not just "target-API floor is 35").
- If you find a fact here contradicted by current official docs, fix the knowledge doc in the same
  session, bump its `last reviewed` date, and note the correction in
  `10-releases/FRAMEWORK_CHANGELOG.md`.
