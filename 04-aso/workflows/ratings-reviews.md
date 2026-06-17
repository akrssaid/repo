# Workflow — Ratings & Reviews Management

This workflow runs the target app's rating and review program: the reply doctrine and SLA tiers,
reply templates keyed to failure labels, escalation of product-bug themes into the backlog, the
in-app review API implementation spec, own-app review mining on a cadence, and a rating-recovery
program with a measured target. It is consumed by `01-core/prompts/12-aso-audit.md` (reads the
review-theme picture) and feeds the **Review-Response Readiness** section of the ASO handover
(`05-handover/ASO_HANDOVER_SCHEMA.md`, D16). Rating-as-conversion theory and the in-app-review
policy line are canonical in `08-knowledge/aso/conversion-optimization.md` §6 — this workflow is
the execution procedure for that doctrine. Conventions (IDs, scales, paths) live in
`01-core/CONVENTIONS.md`.

> **Estimate discipline.** Rating-recovery targets and time-to-recover projections are planning
> **estimates** — label them `(est.)`. Actual rating average/count and reply-SLA compliance are
> measured from the console; record those as measured.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass (P7) — establish doctrine, baseline, and the in-app review trigger | Full workflow |
| Ongoing post-launch (weekly reply queue, cadence mining) | Stages 1, 4 per `04-aso/workflows/post-launch-monitoring.md` |
| Rating drop flagged by post-launch monitoring | Stage 5 rating-recovery program |
| A quality release ships (recovery is most effective right after) | Stage 5 recovery + Stage 3 in-app prompt review |
| In-app review prompt added or changed | Stage 3 implementation + policy re-check |

## Required Inputs

- Console access: Play Console Ratings & reviews; AppGallery Connect; RuStore console.
- `docs/factory/audits/ASO_AUDIT.md` — rating baseline and review-theme findings (the only
  ASO_AUDIT path; `01-core/CONVENTIONS.md`).
- The shared review-failure taxonomy from `04-aso/workflows/competitor-analysis.md` step 6
  (reused here against the target app's own reviews).
- `docs/factory/PROJECT_STATE.md` — to route product-bug themes into Open Questions / Backlog.
- App source access (for Stage 3, the in-app review API integration).

## Procedure

### Stage 1 — Reply Doctrine & SLA Tiers

1. **Reply SLA tiers** (the standing commitment; track compliance as a measured metric):
   | Tier | Review class | SLA | Coverage |
   |---|---|---|---|
   | T1 | 1–2★ | **Reply within 72h** | Every review |
   | T2 | 3★ | Within ~1 week | Every review with substantive text |
   | T3 | 5★ | Best-effort | **Sampled** — reply to a sample that organically carries keywords, to reinforce them |
   1–2★ replies are the program's core: users can revise their rating after a reply, and a visible
   reply pattern lifts conversion for review readers (`08-knowledge/aso/conversion-optimization.md`
   §6).
2. **Reply principles.** Fix-or-explain, never argue; one reply per review; acknowledge the
   specific complaint (no copy-paste that ignores what they said); never request a rating change in
   the reply (policy); never disclose personal data; move debugging to a support channel.
3. **Triage each review to a failure label** using the shared taxonomy from
   `04-aso/workflows/competitor-analysis.md` step 6: `crash/bug`, `ads-density`, `paywall-surprise`,
   `missing-feature`, `privacy/permissions`, `login-forced`, `performance`, `support-silence`,
   `other`. The label selects the reply template and feeds the cadence mining (Stage 4).

### Stage 2 — Reply Templates per Failure Label

4. Maintain a reply template per label; personalize the first clause to the specific review before
   sending. Templates are starting points, not scripts — a visibly templated reply reads worse than
   none. Examples (adapt tone to the brand voice):
   | Label | Template shape |
   |---|---|
   | `crash/bug` | "Sorry this crashed on [device/flow]. We've logged it — a fix is targeted for [version]. If you can share steps at [support], we'll prioritize it." |
   | `performance` | "Thanks for flagging the slowness on [screen]. We're profiling startup/[flow] now and improving it in an upcoming release." |
   | `missing-feature` | "Good call — [feature] isn't in yet. We've added it to our backlog and will update here when it ships." |
   | `paywall-surprise` | "Sorry the paywall wasn't clear. [What's free vs paid]. We're making the boundary clearer in the listing and onboarding." |
   | `ads-density` | "Thanks — ad frequency feedback noted. We're reviewing placements; [remove-ads option if any]." |
   | `privacy/permissions` | "We only request [permission] for [purpose] and don't [share/sell]. Details: [privacy link]. Happy to clarify any concern." |
   | `login-forced` | "You can use [core feature] without an account; sign-in is only for [sync/feature]. If that wasn't clear, we'll fix the prompt." |
   | `support-silence` | "Sorry for the wait. Reaching out now at [channel] to get this resolved." |
5. **Escalate product-bug themes, do not just reply.** When a label recurs (≥3 reviews in a window,
   or any crash theme), open a `RISK`/`Backlog` item: log it under **Open Questions** or **Backlog**
   in `docs/factory/PROJECT_STATE.md` (canonical section names per `01-core/CONVENTIONS.md` / D6)
   with the label, frequency, and a verbatim excerpt. Reviews are free product telemetry — the reply
   handles the user; the escalation handles the cause.

### Stage 3 — In-App Review API Implementation Spec

6. **Use the official in-app review API** (Play In-App Review / `ReviewManager`; AppGallery and
   RuStore have their own in-app review entry points). Never build a custom rating dialog that
   deep-links to the store store-rating page as the primary ask — the official flow keeps the user
   in-app and is the only compliant path.
7. **Trigger conditions — only after a success moment.** Fire the request after a *completed core
   action* (file recovered, document saved, scan exported, task finished) — **never on app launch,
   never mid-task, never after an error, never after an ad**.
8. **Session/action thresholds (est., tune per app):** wait at least **2–3 sessions** AND **≥1
   completed core action** before the first ask; never ask twice in the same session; space repeat
   eligibility by weeks, not days.
9. **Quota behavior.** The API is quota-limited and may show nothing — the request can silently
   no-op. Therefore: **never build UI that promises a dialog**, never block or gate app flow on the
   result, and never branch on whether the dialog appeared. Call it and continue regardless.
10. **The policy line on rating-gating (what is allowed).** Per
    `08-knowledge/aso/conversion-optimization.md` §6: a neutral feedback-first branch ("Enjoying the
    app?" → unhappy users to a feedback form, happy users to the review flow) is currently tolerated,
    but the store policy **bars incentivizing or manipulating reviews**, and gating **must never
    block, discourage, or add friction to the option to review**. Do not show the review prompt only
    to users you've pre-classified as happy via the official API path. When in doubt, take the
    conservative reading and verify against the current Ratings & Reviews policy text.

### Stage 4 — Own-App Review Mining on a Cadence

11. **Reuse the competitor tagging method against the target app.** On a cadence (weekly triage of
    new reviews; a fuller mining pass monthly — see `04-aso/workflows/post-launch-monitoring.md`),
    run `04-aso/workflows/competitor-analysis.md` step 6 on the target app's own 1–3★ reviews: read
    20–30 recent, tag each with the shared label, rank by frequency, keep dated verbatim excerpts.
12. **Route themes to the product backlog.** Rank the labels; for any theme above the escalation
    threshold (Stage 2 step 5), confirm a Backlog/Open Questions entry exists in
    `docs/factory/PROJECT_STATE.md`. Mining without routing is wasted — the deliverable is a
    backlog item with evidence, not a list.
13. **Record the theme snapshot** in `docs/factory/audits/ASO_AUDIT.md` (own-app review-theme
    table) so the audit and handover read from one source.

### Stage 5 — Rating-Recovery Program

14. **Baseline.** Record current rating average and count, recent-window average (last 30/90 days),
    and the dominant complaint labels from Stage 4 in `docs/factory/audits/ASO_AUDIT.md`.
15. **Target (est.).** Set a recovery target tied to a conversion threshold (e.g. cross from 3.9 to
    4.2★ — the rating-display jump that moves CTR materially). Label the target and the projected
    time-to-recover `(est.)`.
16. **Mechanism.** Recovery is: ship the quality fixes that the mined themes pointed to, THEN run
    disciplined success-moment in-app prompting (Stage 3) so recent ratings skew positive. Play
    weights recency — a bad legacy average can recover in one or two quarters of disciplined
    prompting after a quality release (`08-knowledge/aso/conversion-optimization.md` §6). Prompting
    before fixing just harvests more low ratings.
17. **Measurement window.** Review at +30 and +90 days: recent-window average, count velocity, and
    label-frequency shift. If the recent average is not trending toward target after one quarter,
    the cause is unfixed product issues, not prompting cadence — return to Stage 4 routing.

### Stage 6 — Per-Store Reply Mechanics

18. Reply mechanics differ per store (consult the store guides for current console steps):
    - **Play** — Play Console → Ratings & reviews → Reviews; filter by rating, reply inline; users
      are notified and can revise. See `04-aso/stores/google-play.md`.
    - **AppGallery** — AppGallery Connect → Operate → User reviews; reply per review. See
      `04-aso/stores/huawei-appgallery.md`.
    - **RuStore** — RuStore console reviews section; reply in Russian (match the listing locale). See
      `04-aso/stores/rustore.md`.
    Track per-store reply-SLA compliance separately; AppGallery/RuStore notification behavior differs
    from Play.

## Outputs

| Artifact | Path | Notes |
|---|---|---|
| Reply doctrine, SLA tiers, templates | `04-aso/workflows/ratings-reviews.md` (this file, applied) + handover | — |
| Review-Response Readiness section | `docs/factory/handovers/ASO_HANDOVER_v<N>.md` | Per `05-handover/ASO_HANDOVER_SCHEMA.md` (D16) |
| Own-app review-theme snapshot + rating baseline/target | `docs/factory/audits/ASO_AUDIT.md` | Target labeled `(est.)` |
| Product-bug escalations | `docs/factory/PROJECT_STATE.md` (Open Questions / Backlog) | Canonical section names, D6 |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` | — |

## Acceptance Criteria

- [ ] Reply SLA tiers defined and tracked: every 1–2★ replied within 72h; 5★ sampled for keyword
      reinforcement; compliance measured from the console.
- [ ] Every reply triaged to a shared failure label; recurring product-bug themes escalated to
      `docs/factory/PROJECT_STATE.md` Open Questions / Backlog with evidence.
- [ ] In-app review API integrated via the official flow; triggers fire only after a success moment;
      session/action thresholds set; no UI promises a dialog; rating-gating respects the policy line.
- [ ] Own-app review mining run on a cadence using the shared taxonomy; themes routed to the backlog,
      snapshot recorded in `docs/factory/audits/ASO_AUDIT.md`.
- [ ] Rating-recovery program has a baseline, an `(est.)`-labeled target tied to a conversion
      threshold, the fix-then-prompt mechanism, and a +30/+90-day measurement window.
- [ ] Per-store reply mechanics documented; Review-Response Readiness section produced for the ASO
      handover.
