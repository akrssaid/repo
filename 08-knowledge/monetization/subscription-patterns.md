# Subscription & IAP Patterns for Utility Apps

Evergreen doctrine for paid monetization of utility apps: when subscriptions fit, how to build
paywalls that convert without dark patterns, and the Play Billing / RuStore / HMS rail facts that
matter. Agents consult this during P3 (audit existing monetization), P4/P6 (implementation), and
P7 (paywall claims must match listing claims). Companion file: ad monetization in
`08-knowledge/monetization/ads-patterns.md`; multi-store billing rails in
`08-knowledge/stores/store-comparison.md`.

**Last reviewed: 2026-06-10.** Billing library minimum-version mandates, service-fee tiers,
subscription policy text, and RuStore/HMS SDK capabilities all drift — verify against current
official Play Billing, AppGallery Connect, and RuStore docs before implementation. All
benchmark figures are labeled estimates.

---

## 1. Subscription vs one-time unlock — the recurring value test

Ask: **does the paid value recur without the user doing anything new?**

| Signal | Points to |
|---|---|
| Ongoing service cost you bear (cloud storage/sync, server-side processing, API costs) | Subscription |
| Continuous content/updates users actually feel (new formats, ongoing OCR-model improvements) | Subscription |
| Static feature set: dark theme, batch mode, remove-ads, pro filters | One-time (lifetime) unlock |
| Mixed: static features + one recurring service | Hybrid menu |

Doctrine for utility apps specifically:

- Most utility-app premium value is **static** — a subscription on a static feature set reads as
  rent-seeking, drives 1★ reviews ("why is a flashlight a subscription"), and churns at M1.
- **The hybrid menu is the pragmatic default**: offer monthly / yearly / lifetime side by side.
  It lets subscription-tolerant users subscribe and subscription-averse users (a large share of
  utility audiences) pay once — capturing both instead of arguing with either.
- **Price-ratio guidance (estimate, tune per market)**: monthly : yearly : lifetime ≈
  **1 : 6–8 : 20–25**. Example: $2.99 / $19.99 / $59.99. Yearly at 6–8 monthly-equivalents makes
  yearly the obvious "smart buy"; lifetime at ~2.5–3× yearly monetizes the averse without
  cannibalizing yearly. Re-derive from your own M1 churn once data exists (§6).
- Remove-ads as an SKU and its price anchoring: see
  `08-knowledge/monetization/ads-patterns.md` §6 — in a hybrid menu it usually merges into the
  pro unlock rather than standing alone.

## 2. Paywall doctrine

**Placement — after value demonstration, never before:**

1. Never hard-paywall the first launch. The user must complete (or visibly preview) one core
   task before any paywall — a paywall before value is an uninstall generator and inflates the
   misleading-listing risk (`08-knowledge/stores/policy-landmines.md` §1).
2. Trigger paywalls at **capability boundaries**: user attempts a pro feature, hits a free-tier
   quota, or finishes a success moment where the pro upsell is contextually obvious.
3. Free-tier design: **prove value, reserve headroom.** Free must fully solve the basic job
   (else ratings die); pro must own the depth (batch, automation, large files, no ads, cloud).
   Quota-based free tiers (N operations/day) suit recurring-use utilities; feature-gated free
   tiers suit static ones.

**Trial mechanics:**

- **7-day free trial with a pre-renewal reminder is the highest-integrity pattern** — users who
  convert from a reminded trial refund less, churn less, and review better. Use Play's built-in
  trial offers; never fake a "trial" by delaying a charge manually.
- Trial requires payment method up front on Play — design the paywall copy to say exactly when
  billing starts ("Free for 7 days, then $19.99/yr — cancel anytime"). Ambiguity here is both a
  dark pattern (§5) and a Play subscriptions-policy violation.

**Paywall anatomy (single-decision screen):**

- **Benefit list, not feature list**: "Recover unlimited files" not "Unlimited mode"; 3–5
  benefits max, each one line with a check glyph.
- One screen, one decision: plan selector (the hybrid menu), one primary CTA, price + renewal
  terms in plain text adjacent to the CTA — not buried in footnote gray.
- **Restore purchases visible** on the paywall itself (and in settings) — required by user
  expectation and effectively by store review; hiding it generates support load and refunds.
- Close affordance present and immediate (a paywall that traps = dark pattern + review risk).
- Yearly pre-highlighted is acceptable; pre-selected *without* clear per-period pricing is not (§5).

## 3. Play Billing essentials

- **Billing Library version currency**: Google **enforces minimum Billing Library versions** —
  new submissions and updates are blocked below the floor (historically: each major version is
  accepted ~2 years; e.g., the v6 floor in 2024, v7 in 2025 cycle). Check the current minimum in
  official docs at every release; treat an aging billing dependency as a Critical roadmap item
  in P3 audits (`02-engineering/03-dependency-analysis.md`).
- **Base plans + offers model** (Billing v5+): one subscription product carries multiple base
  plans (monthly, yearly) and offers (free trial, intro price, win-back) layered per base plan.
  Model the hybrid menu as: one sub product with monthly+yearly base plans, plus a separate
  one-time in-app product for lifetime.
- **Acknowledge every purchase** within 3 days or Play auto-refunds it — acknowledge
  server-side or in `BillingGateway` immediately after grant.
- **Grace period and account hold**: enable both in Console; in-app, detect the states and show
  a fix-payment banner rather than yanking entitlement on day one — grace-period users recover
  at high rates when prompted gently.
- **Obfuscated account ID**: pass `setObfuscatedAccountId` (a salted hash of your internal user
  id) on every billing flow — it is your primary tool for tracing **refund-abuse patterns**
  (serial refund-and-keep-using) and for support lookups via Play's order reports.
- Server-side verification (Play Developer API purchases/subscriptions v2 + Real-Time Developer
  Notifications via Pub/Sub) is the correctness baseline for any app with accounts or sync;
  pure-local utility apps may verify locally but must then treat entitlements as
  device-scoped honestly.

## 4. Alternative rails (multi-store)

| Capability | Play Billing | RuStore pay SDK | HMS IAP |
|---|---|---|---|
| Subscriptions | Full (base plans/offers, proration, pause) | Supported; simpler model — verify current capability matrix | Supported (subscription IAP type) |
| Free trials / intro pricing | Native offer types | Partial — confirm per current SDK docs | Supported with config in AG Connect |
| Server verification | Developer API + RTDN push | REST API + webhook callbacks — design webhook ingestion early; polling is the fallback | Order/subscription verification REST API |
| Refund visibility | Voided purchases API | Via console/API | Via AG Connect |
| Payment methods | Cards/wallets/carrier | MIR cards, SBP, RF mobile carriers | Region-dependent Huawei rails |

Engineering doctrine: one `BillingGateway` interface, three flavor implementations, single local
entitlement store — detailed in `08-knowledge/stores/store-comparison.md` §3. Capability gaps
(e.g., a trial type missing on one rail) must degrade the *offer*, never fork the entitlement
model. Keep paywall copy per-flavor accurate — advertising a trial that the RuStore rail can't
deliver is a moderation rejection there.

## 5. Dark-pattern ban list (ethics + enforcement aligned)

Play's subscriptions policy explicitly requires clear disclosure of terms, price, and renewal —
and enforcement has teeth. Banned in factory-built apps regardless:

| Dark pattern | Why banned |
|---|---|
| Forced trial with hard cancel (signup in 1 tap, cancellation hidden/multi-step/external) | Play requires easy cancellation access; generates refunds + 1★ floods |
| Fake discounts ("90% off!" from a never-charged anchor price) | Deceptive pricing; metadata/claims violation territory |
| Pre-selected long plan without per-period clarity (yearly preselected showing only "$0.38/day") | Disclosure policy: full billed amount + period must be unambiguous before purchase |
| Countdown timers on paywalls that reset | Manufactured urgency = deceptive behavior |
| Trial-end silence by design (relying on users forgetting) | Play sends its own reminders anyway; designing against them erodes trust and refunds spike |
| Quota shrinking post-install (free tier silently reduced to force upgrades) | Listing described the free product; this is bait-and-switch |
| Double-paywall (paywall immediately after dismissing a paywall) | Disruptive behavior; uninstall driver |

The commercial argument, not just the moral one: dark-pattern revenue arrives with refunds,
chargebacks, policy strikes, and rating collapse — it is **negative-LTV revenue** once the
ranking flywheel damage is priced in (`08-knowledge/aso/ranking-factors.md` §7).

## 6. Retention, churn, and lifecycle

- **Cancel-flow win-back (policy-safe versions)**: when a user heads to cancel, you may show a
  one-screen offer (pause, downgrade to a cheaper base plan, or a win-back discount offer
  configured in Play Console) — **one screen, with the cancel path still one tap away**.
  Obstructing cancellation is the line; offering an alternative once is fine.
- **Price increases**: use Play's price-change flow — existing subscribers must be notified and
  (for opt-in-required changes) explicitly consent, else they churn at renewal per current
  policy mechanics; verify the current opt-in vs opt-out rules in official docs. Grandfathering
  legacy prices for loyal cohorts is usually cheaper than the churn from migrating them.
- **Refund posture**: approve fast, no-questions, within reason — fighting small refunds buys
  1★ reviews and chargebacks. Track repeat abusers via obfuscated account id (§3) and gate
  re-purchase server-side rather than arguing individual cases.

## 7. Metrics (labeled estimates — calibrate per app)

| Metric | Definition | Weak | Typical (utility) | Strong |
|---|---|---|---|---|
| Paywall view → trial start | Of paywall viewers, % starting trial | <2% | 4–8% | >12% |
| Trial → paid conversion | Of trial starters, % converting at first charge | <20% | 30–45% | >55% |
| M1 subscriber churn | % of paying subs lost in first renewal month | >25% | 10–18% | <8% |
| Yearly share of new subs (hybrid menu) | % choosing yearly | <30% | 40–60% | >65% |
| Lifetime share of paid revenue (hybrid) | % of revenue from lifetime SKU | — | 20–40% | context-dependent |
| Refund rate | Refunds / purchases | >5% | 1–3% | <1% |

Instrument: paywall impressions (with trigger source), plan selection, trial starts, conversions,
cancellations with survey reason — into whatever analytics the app uses, reviewed at every P3
re-audit. A rising paywall-view count with flat trial starts means a placement problem (§2), not
a pricing problem; exhaust placement and copy changes before discounting.

## 8. Cross-references

- Ads counterpart and remove-ads anchoring: `08-knowledge/monetization/ads-patterns.md`
- Billing rails per store + flavor architecture: `08-knowledge/stores/store-comparison.md`
- Policy enforcement detail: `08-knowledge/stores/policy-landmines.md`
- Dependency currency audits: `02-engineering/03-dependency-analysis.md`
- Listing-claims alignment: `08-knowledge/aso/conversion-optimization.md`
- Release gate: `07-checklists/release-readiness.md`
