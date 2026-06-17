# Ads Monetization Patterns for Utility Apps

Evergreen doctrine for monetizing utility apps with ads without destroying retention, ratings, or
policy standing. Agents consult this during P3 (audit — is the current ad implementation hurting
the app?), P4/P6 (implementation), and P7 (the ads↔ASO interlock: ad UX drives uninstall velocity
and ratings, which drive rank — see `08-knowledge/aso/ranking-factors.md` §3). Companion file for
paid monetization: `08-knowledge/monetization/subscription-patterns.md`.

**Last reviewed: 2026-06-10.** Network SDKs, eCPM levels, consent law, and ad-policy enforcement
drift constantly — verify against current AdMob/AppLovin/Yandex official docs and the Play
Families/Ads policies before implementing. All revenue figures here are labeled estimates.

---

## 1. Network landscape and compatibility

| Network | GMS Play build | AppGallery (GMS-less) | RuStore (RU users) | Role |
|---|---|---|---|---|
| AdMob | ✅ primary | ❌ requires GMS | ⚠️ loads but RU demand/payouts blocked | Default primary network on Play |
| AdMob mediation | ✅ | ❌ | ⚠️ | First mediation step when scale justifies |
| AppLovin MAX | ✅ | ✅ (GMS-independent) | ⚠️ partial RU demand | Mediation alternative; strong rewarded/interstitial demand |
| Yandex Ads (Mobile Ads SDK) | ✅ | ✅ | ✅ primary | The monetizing network for RU/RuStore builds |
| HMS Ads Kit | ❌ pointless | ✅ primary | ❌ | AppGallery flavor network |

Doctrine:

- **Single network first.** AdMob alone until the app clears roughly $50–100/day ad revenue
  (estimate) — mediation adds SDK weight, init latency, and debugging surface that isn't paid
  for at small scale.
- **Mediation when scale justifies**: AdMob mediation (stay in Google tooling) or migrate to
  AppLovin MAX (better if rewarded volume is high or AG builds matter). Run one mediation stack,
  never two.
- **Per-store flavors carry their network** via an `AdsFacade` abstraction — flavor strategy and
  capability map in `08-knowledge/stores/store-comparison.md` §3/§6.
- Both MAX and AdMob **require `app-ads.txt`** on the developer website domain listed in the
  store listing; missing/incorrect entries silently crater fill. Verify file contents against
  each network's current official spec.

## 2. Format playbook

### Banner
- **Anchored adaptive banner** at screen bottom: the workhorse. Acceptable on list/browse screens
  and results screens; never on screens requiring precision input near the bottom edge.
- **List-native style placement** (banner or native in list flow): allowed every N≥8 items,
  clearly distinct from content rows.
- Where banners hurt: reading/viewing surfaces (document pages, video playback), input forms,
  and any screen where the banner would push a primary action button. Collapsible banners are
  under elevated AdMob policy scrutiny — implement only per current official guidance.

### Interstitial — the retention killer when misused
- **ONLY at natural task boundaries**: after-save, after-export, after-task-complete, on
  back-navigation from a finished flow. The user's intent loop must already be closed.
- **Never between intent and result**: not after tapping "Recover"/"Download"/"Open" and before
  the outcome appears. That placement is the single most rating-destructive pattern in utility
  apps and borders on Play's disruptive-ads policy.
- **Frequency caps (doctrine, tune per app)**: never in the first session; no interstitial in
  the first ~60–90 s of any session; minimum gap 1 per 2–3 minutes; daily cap (e.g., ≤6–10).
  Enforce caps client-side in the `AdsFacade`, not by hoping the network's defaults align.
- Always preload; a spinner waiting for an ad is a double offense.

### Rewarded
- **Unlock-feature pattern** for utility apps: one premium-tier action (batch export, HD mode,
  extra theme) granted per rewarded view. Opt-in by definition, so it's the highest-eCPM,
  lowest-resentment format.
- Never gate core advertised functionality behind rewarded ads — that converts the store
  listing into a misleading claim (`08-knowledge/stores/policy-landmines.md` §1).
- Rewarded interstitials must keep the pre-ad opt-out prompt that the format mandates.

### App-open
- **Cold start only**, with a cap (e.g., ≤1 per N hours), never on warm foreground from a
  notification tap or in-flow returns from pickers/share sheets (classic accidental-violation
  path: the file picker round-trip fires app-open ads mid-task).
- Never on first launch ever (the first-run experience belongs to onboarding).

### Native
- Integration rule: native ads in lists must keep the platform-mandated **ad attribution label
  and AdChoices icon visible**, must not mimic content rows pixel-for-pixel, and must not sit
  adjacent to click-bait-styled content. Headline/CTA must come from the network assets
  unmodified. Deceptive native styling = ads policy strike.

## 3. Format × surface decision table

| Surface | Recommended | Forbidden |
|---|---|---|
| Home / dashboard | Anchored banner or none | Interstitial on entry |
| List / browse | Anchored banner; native every ≥8 rows | App-open on return from item |
| Task in progress | Nothing | Everything |
| Task result / success screen | Interstitial (capped) after result rendered; rewarded upsell | Ad before result renders |
| Settings / about | None (zero revenue, pure annoyance) | All |
| Viewer / reader / player | None, or banner outside content area | Overlay ads on content |

## 4. UX guardrails (policy-interlocked)

- **Cross-reference** `08-knowledge/design/mobile-ux-principles.md` ad-coexistence rules — ads
  are part of the design system, not a layer dropped on top.
- **Layout-shift ban**: reserve banner space at layout time (fixed-height container) so content
  never jumps when fill arrives. Layout shift causes accidental clicks, and **accidental-click
  patterns are AdMob policy strikes** (invalid traffic) as well as Play disruptive-ad
  violations — this is the guardrail with teeth on both sides.
- No ads within 16dp-thumb-radius of frequent tap targets; no ads that appear under a finger
  mid-gesture; close buttons at full mandated size, never delayed-reveal beyond network spec.
- Test every ad surface with fill disabled (no-fill is the most common production state in low
  eCPM geos) — the UI must be complete without the ad.

## 5. Revenue math literacy

All figures are **rough 2026 planning estimates, blended and volatile** — actual eCPM varies 10×
by geo, season (Q4 ≈ 1.5–2× Q1), and demand. Never promise these numbers.

| Format | Tier-1 geo eCPM (est.) | Tier-3 geo eCPM (est.) | Typical fill reality |
|---|---|---|---|
| Anchored banner | $0.3–1.5 | $0.05–0.3 | High fill, low value |
| Interstitial | $5–15 | $0.5–2 | Good fill; capped by your frequency rules |
| Rewarded | $10–30 | $1–4 | Opt-in volume limits revenue |
| App-open | $3–8 | $0.3–1.5 | One impression/session by design |
| Native | $1–5 | $0.2–1 | Placement-quality dependent |

**Why retention beats ad density** — the LTV sketch every decision should run through:

```
ad LTV ≈ Σ over days d: retention(d) × sessions/day × impressions/session × eCPM/1000
```

Doubling impressions/session raises today's term but cuts retention(d) for all future d; in
utility apps an aggressive interstitial load reliably costs more D7/D30 area-under-curve than it
adds per-session revenue — and the retention loss also feeds the ranking flywheel negatively
(`08-knowledge/aso/ranking-factors.md` §7), shrinking the install base itself. Tune ad load to
maximize the integral, not the day-1 ARPDAU, and watch uninstall velocity + rating recency as
the early-warning pair after every ad-load change.

## 6. Ads + IAP hybrid

- **Remove-ads SKU is mandatory doctrine** for any ad-funded utility: it monetizes exactly the
  users ads monetize worst (high-intent, ad-averse), and its price anchors the premium tier.
- Price using the LTV sketch: remove-ads should cost ≥ 6–12 months of that user's expected ad
  LTV (estimate) — typically lands at a $2.99–6.99 one-time or folds into the pro/lifetime menu
  (`08-knowledge/monetization/subscription-patterns.md` §1).
- Surface the offer at ad-friction moments (after an interstitial closes — "tired of ads?"),
  never as an interruption of its own. Conversion on remove-ads concentrates in the first week
  and after ad-load increases — which is a signal to read, not a lever to abuse.
- Entitlement must kill **all** ad code paths immediately and persistently (local DataStore
  flag checked by `AdsFacade` before any load call) — a purchased user seeing one more ad is a
  guaranteed 1★ + refund.

## 7. Policy interlocks (the non-negotiables)

1. **Consent (UMP)**: EEA/UK (+ Switzerland) traffic requires a certified CMP — Google's UMP SDK
   is the default; gather consent before any ad request in scope, propagate consent state to
   mediation partners. Misconfigured consent = unfilled EEA inventory at best, regulatory
   exposure at worst. Verify current UMP integration docs.
2. **Families policy**: if the app's declared audience includes children, only certified ad
   SDKs/configs are allowed and personalization is restricted — utility apps should declare an
   accurate adult/general audience (`08-knowledge/stores/policy-landmines.md` §4) and still set
   appropriate content ratings on ad requests.
3. **`app-ads.txt`** for AdMob and MAX both, on the developer domain in the store listing (§1).
4. **Fake-system-UI and disruptive-ad bans** are Play policy, enforced independently of the ad
   network's own rules — passing AdMob review does not imply passing Play review
   (`08-knowledge/stores/policy-landmines.md` §3).
5. Per-store consent/privacy declarations (Play Data safety must declare ads SDK collection;
   AG/RuStore equivalents) regenerate whenever the ad stack changes
   (`02-engineering/04-sdk-inventory.md`).

## 8. Cross-references

- Subscriptions/IAP counterpart: `08-knowledge/monetization/subscription-patterns.md`
- Multi-store ads compatibility: `08-knowledge/stores/store-comparison.md`
- UX coexistence rules: `08-knowledge/design/mobile-ux-principles.md`
- Ranking impact of ad UX: `08-knowledge/aso/ranking-factors.md`
- Policy detail: `08-knowledge/stores/policy-landmines.md`
- Release gate: `07-checklists/release-readiness.md`
