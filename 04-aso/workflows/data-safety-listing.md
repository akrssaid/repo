# Workflow — Data-Safety / Privacy Declaration

This workflow produces the store data-safety and privacy declarations from the app's actual SDK
footprint, keeps them honest against real network behavior, and re-derives them whenever a
dependency changes. It maps the SDK inventory to each store's data-safety answers (Play Data safety
form; AppGallery and RuStore privacy declarations), using the reusable common-SDK mapping table.
It feeds the **Data-safety** dimension of `01-core/prompts/12-aso-audit.md` and the
`07-checklists/aso-launch.md` gate. Conventions and canonical paths are in `01-core/CONVENTIONS.md`.

> The data-safety form is a **compliance artifact, not a marketing one** — there is no estimate
> latitude. Every answer must match what the app and its SDKs actually do. A form that overstates is
> a bad look; a form that understates is an **enforcement risk** (removal/suspension). When unsure,
> verify the SDK's documented behavior and observe traffic — do not guess.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO/launch pass (P7/P8), before submission | Full workflow, all target stores |
| **Any dependency change** (add/remove/upgrade an SDK) | Full re-derivation — see re-run triggers |
| `02-engineering/03-dependency-analysis.md` reports a new direct/transitive SDK | Stages 1–2 for the new SDK, then re-submit |
| Store policy update to the data-safety/privacy questionnaire | Re-map affected answers |
| Pre-release verification (`01-core/prompts/16-release-preparation.md`) | Consistency check (Stage 5) |

## Required Inputs

- `docs/factory/audits/ENGINEERING_AUDIT.md` SDK inventory section, produced by
  `02-engineering/04-sdk-inventory.md` (per-SDK dependency + manifest + init evidence).
- Dependency graph dumps `docs/factory/audits/deps-<config>.txt` from
  `02-engineering/03-dependency-analysis.md` (the change-detection source).
- The reusable common-SDK mapping table in `08-knowledge/stores/data-safety-mapping.md`.
- The app's privacy policy URL and any in-app data collection (forms, accounts, analytics events).
- Store consoles: Play Console (App content → Data safety), AppGallery Connect, RuStore console.

## Procedure

### Stage 1 — SDK Inventory → Data-Safety Mapping

1. **Start from the SDK inventory.** Take every third-party SDK from the inventory produced by
   `02-engineering/04-sdk-inventory.md` (ads, analytics, crash reporting, attribution, billing,
   push). Include SDKs present in the dependency graph even if they never initialize — they ship
   manifest entries and can still collect data; the form covers what *ships*, not only what runs.
2. **Map each SDK to data-safety rows.** For each SDK, produce a row answering the store's core
   questions:
   | Column | What to record |
   |---|---|
   | SDK | name + artifact id |
   | Data types | which data types it touches (device/IDs, location, app activity, crash logs, contacts, etc.) |
   | Collected? | does the app/SDK **collect** (leave the device) this data |
   | Shared? | is it **shared** with a third party (Play distinguishes collected vs shared) |
   | Purpose | analytics / advertising / app functionality / fraud-prevention / personalization |
   | Processed ephemerally? | processed in memory and not retained |
   | Optional? | user can opt out / required for the feature |
3. **Use the common-SDK mapping table as the reusable source.** Do not re-derive well-known SDKs
   (Firebase Analytics, Crashlytics, AdMob, AppLovin, etc.) by hand — read their canonical
   collection/sharing/purpose rows from `08-knowledge/stores/data-safety-mapping.md` (that doc holds
   the reusable table; cite it, don't restate it here). Hand-derive only SDKs not in that table, and
   the app's own first-party collection (accounts, support forms, in-app analytics events).
4. **Add first-party collection.** Account creation, support/contact forms, user-generated content,
   and any in-app analytics the app itself sends are collection too — add rows for them.

### Stage 2 — Resolve the Form Answers

5. **Aggregate SDK rows into per-data-type answers.** The store form asks per data type, not per
   SDK: a data type is "collected" if *any* shipped SDK or first-party path collects it; "shared" if
   *any* shares it. Roll the Stage 1 rows up into one answer per data type, keeping the SDK rows as
   the evidence/audit trail behind each answer.
6. **Map purposes and optionality** per data type from the rolled-up SDK purposes. Where one data
   type serves multiple purposes (e.g. an advertising ID used for both ads and analytics), declare
   all applicable purposes.

### Stage 3 — Play Console Data Safety Form Submission

7. In Play Console → **App content → Data safety**, complete the questionnaire from the Stage 2
   answers:
   1. Declare whether the app collects/shares data; then per data type set collected/shared,
      purpose(s), optional vs required, and ephemeral processing.
   2. Set the privacy-policy URL (required when any data is collected).
   3. Declare security practices (encryption in transit; data-deletion request path) truthfully.
   4. Review the generated data-safety summary against the Stage 1 evidence table before submitting.

### Stage 4 — AppGallery / RuStore Privacy Declarations

8. **AppGallery** — complete the privacy-declaration / data-processing fields in AppGallery Connect
   and attach the privacy policy; Huawei reviews privacy declarations as part of listing approval.
   See `04-aso/stores/huawei-appgallery.md` for the current field-by-field mechanics.
9. **RuStore** — complete RuStore's privacy/data-handling declaration and policy URL fields. See
   `04-aso/stores/rustore.md`. Derive both from the **same** Stage 2 answer set — divergent
   declarations across stores for the same build are a red flag.

### Stage 5 — Consistency Rule (form must match actual traffic)

10. **The form must match what the app actually does on the wire.** Verify the declaration against
    real behavior before submission:
    1. Cross-check each "collected/shared" answer against the SDK's documented behavior and the
       manifest evidence from the inventory.
    2. Where feasible, observe actual network traffic on a debug build (proxy/network inspector) and
       confirm no undeclared endpoint is sending a declared-absent data type.
    3. Any mismatch (form says not-collected, traffic shows collection — or vice versa) is resolved
       **before** submission. Mismatch between the declared form and actual SDK traffic is an
       enforcement trigger (rejection, removal, or suspension), not a cosmetic issue.

**Worked shape (illustrative — read real values from the mapping table).** A typical utility app
    bundling Firebase Analytics + Crashlytics + AdMob rolls up to roughly: *App activity* and
    *App info & performance* (crash logs) **collected** for analytics + app-functionality;
    *Device or other IDs* (advertising ID) **collected and shared** for advertising; location only if
    a location-using SDK or feature is present. The exact collected/shared/purpose values for each of
    those SDKs come from `08-knowledge/stores/data-safety-mapping.md` — this shape is only to orient
    the reviewer, never a substitute for the table.

### Stage 6 — Re-Run Triggers

11. **Re-derive on any dependency change.** Adding, removing, or upgrading any SDK can change what
    data ships — so any change reported by `02-engineering/03-dependency-analysis.md` (new direct or
    transitive SDK, an upgrade that adds data collection) re-triggers Stages 1–5 for the affected
    SDK and a re-submission. Wire this into the dependency-change review, not a calendar — a silent
    SDK upgrade that starts collecting a new data type with no form update is the classic
    enforcement case.

## Outputs

| Artifact | Path | Notes |
|---|---|---|
| Per-SDK → data-safety evidence table | `docs/factory/audits/ASO_AUDIT.md` (Data-safety section) | Audit trail behind each form answer |
| Rolled-up per-data-type answer set | `docs/factory/audits/ASO_AUDIT.md` | Single source for all stores |
| Submitted Play Data safety form | Play Console | Matches the answer set |
| AppGallery / RuStore privacy declarations | respective consoles | Derived from the same answer set |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` | Date, trigger (launch / dependency change) |

## Acceptance Criteria

- [ ] Every shipped third-party SDK (from `02-engineering/04-sdk-inventory.md`) plus first-party
      collection mapped to data-safety rows; well-known SDKs sourced from
      `08-knowledge/stores/data-safety-mapping.md`, not hand-re-derived.
- [ ] SDK rows rolled up into one answer per data type, with the SDK rows kept as evidence.
- [ ] Play Data safety form submitted from the answer set, with privacy-policy URL and truthful
      security practices.
- [ ] AppGallery and RuStore privacy declarations derived from the **same** answer set (no
      cross-store divergence for the same build).
- [ ] Consistency check done: each answer cross-checked against SDK behavior and, where feasible,
      observed traffic; no mismatch left unresolved at submission.
- [ ] Re-run wired to dependency changes via `02-engineering/03-dependency-analysis.md`; feeds the
      Data-safety dimension of `01-core/prompts/12-aso-audit.md` and the `07-checklists/aso-launch.md`
      gate.
