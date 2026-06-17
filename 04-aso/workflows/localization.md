# Workflow — Localization

This workflow decides which locales a target app should invest in, at what depth, and how to
execute listing and in-app localization to native quality without a translation budget beyond
AI translation plus selective native review. It is consumed by `01-core/prompts/13-keyword-research.md`
(per-locale keyword work), `01-core/prompts/14-store-assets.md` (localized listings), and
`01-core/prompts/16-release-preparation.md` (locale QA gates). Store-specific locale mechanics
live in `04-aso/stores/google-play.md`, `04-aso/stores/huawei-appgallery.md`, and
`04-aso/stores/rustore.md`.

## When to Run

| Trigger | Scope |
|---|---|
| First ASO pass (P7) on any app shipping to RuStore | Stage 1 + RU at Tier 1 minimum (mandatory) |
| Play Console shows ≥10% of installs from a non-localized geo | Stage 1 re-prioritization, then Stages 3+ |
| Quarterly ASO refresh | Stage 7 measurement review; expand/contract tiers |
| Major listing rewrite or screenshot refresh | Stage 3 re-run for all Tier 1+ locales |
| App adds AR/HE or other RTL target | Stage 4 RTL smoke test mandatory |

## Required Inputs

- Play Console acquisition reports by country (current install geography).
- Keyword map (`docs/factory/reports/KEYWORD_MAP.md`) and competitor analysis
  (`docs/factory/reports/COMPETITOR_ANALYSIS.md`) for the base locale.
- Target app source access (for Tier 2+): `res/values/strings.xml` and qualified variants.
- Store distribution plan from `docs/factory/PROJECT_STATE.md` (which stores the app ships on).

## Procedure

### Stage 1 — Locale Prioritization Model

1. Score candidate locales on two axes and rank by the product:
   - **Current install geo**: % of installs and % of store-listing visitors per country from Play
     Console (visitors converting poorly in a non-localized geo = the cheapest win available).
   - **Store opportunity**: category competition in that locale (run a light version of
     `04-aso/workflows/competitor-analysis.md` step 1: how strong are top-10 results for the core
     keyword in that locale?) vs translation cost (listing-only is cheap; in-app strings scale
     with string count).
2. Apply the standing rules:
   - **RU is mandatory for RuStore** — a RuStore listing is submitted in Russian; shipping there
     without native-quality RU listing and (strongly recommended) RU in-app strings is not viable.
     See `04-aso/stores/rustore.md`.
   - **Play candidates to evaluate by category opportunity vs translation cost**: ES (huge
     aggregate reach, LATAM + Spain), PT-BR (large single market, often weak competition),
     ID and TR (high-volume, low-competition growth markets, price-sensitive), DE and FR (high
     revenue per user, but high quality bar — bad German is worse than English in DE).
   - AppGallery skews toward emerging markets and RU/MENA — weight ID/TR/AR/RU higher if the app
     ships there; see `04-aso/stores/huawei-appgallery.md`.
3. Output a ranked locale table with a go/no-go and assigned tier per locale.

### Stage 2 — Scope Tiers

4. Assign each "go" locale a tier; tiers are cumulative:
   | Tier | Scope | Use when |
   |---|---|---|
   | **Tier 1** | Listing only: title, short description, full description, screenshot captions | Default entry point; tests demand cheaply |
   | **Tier 2** | Tier 1 + in-app strings (`strings.xml` and variants) | Locale drives ≥10% installs or shows Tier 1 conversion lift; reviews complain about language |
   | **Tier 3** | Tier 2 + culturalized screenshots/assets (local content in UI shots, locale-appropriate imagery, local payment/brand references) | Strategic locales with proven revenue; RU for RuStore flagship apps |
   A Tier 1 locale whose listing converts but whose reviews then complain the app is English-only
   is the signal to promote to Tier 2.

### Stage 3 — Listing Localization Procedure

5. **Keyword research PER LOCALE — never translate keywords.** Run
   `04-aso/workflows/keyword-research.md` from Stage 1 in the target locale: harvest store
   search-suggest with local-language seed prefixes, score, cluster, slot. Local phrasings differ
   structurally — the high-volume German query is rarely the dictionary translation of the
   high-volume English one, and direct translation routinely targets phrases nobody types.
   Volume/difficulty remain estimates and must be labeled `(est.)`.
6. Localize listing copy against the per-locale keyword map: title carries 1 primary local
   cluster; short description 2–3 secondary; full description natural coverage — same slotting
   rules as the base locale.
7. **Native-quality bar — AI translation + native review fallback rules:**
   1. Draft with AI translation, providing full context (app purpose, audience, character limits,
      keyword slots that must survive intact).
   2. Back-translate to the source language with a separate pass; any meaning drift = redraft.
   3. **Native review is required** for: title and short description in every locale (highest
      exposure, least room for error), all Tier 3 content, and all DE/FR/JA-class
      high-quality-bar locales. Use a native speaker (community, freelancer, or trusted user).
   4. **Fallback when no native reviewer is available**: ship full description and captions from
      the double-checked AI draft, but keep the title conservative (brand + one safe local
      keyword) and flag the locale `pending native review` in the output report — do not ship
      clever wordplay or idiom unreviewed.
8. Localize screenshot captions per `04-aso/workflows/screenshot-optimization.md` Stage 4
   (rewrite for meaning and length, never word-for-word).

### Stage 4 — In-App Strings Procedure (Tier 2+)

9. **Extraction.** Inventory `res/values/strings.xml` plus any per-module string files. Run lint
   to find strings that escaped resources (gradlew; on Windows use `.\gradlew.bat`):
   ```bash
   ./gradlew :app:lintDebug
   ```
   Fix every `HardcodedText` finding before translating — hardcoded strings cannot localize.
10. **Pseudo-locale testing before any real translation.** Enable pseudo-locales on a debug build
    and set device language to `en-XA` (expanded accented English) and `ar-XB` (forced RTL). The
    `settings put system system_locales` route works only where `system` settings are writable —
    it requires a **rooted device or an emulator with writable system settings** (and the change is
    silently ignored on a stock retail device). On any device, the portable path is **Settings →
    System → Languages** (enable Developer options → "Use pseudolocales" first), which needs no root:

    ```bash
    # Rooted device / writable-system emulator only:
    adb shell settings put system system_locales en-XA
    ```

    ```powershell
    # Rooted device / writable-system emulator only:
    adb shell settings put system system_locales en-XA
    ```

    On any stock device, use device **Settings → Languages** (API 26+) instead. `en-XA` lengthens
    strings ~30% and reveals truncation, clipping, and hardcoded text instantly — far cheaper than
    finding it in German.
11. Translate `strings.xml` values via the Stage 3 quality procedure. **Integrity checks on every
    translated file:**
    - **Plurals**: every `<plurals>` carries the quantity classes the target language requires
      (RU needs `one/few/many/other`; EN's `one/other` is insufficient — missing classes cause
      wrong text, not crashes, so check explicitly).
    - **Format args**: every `%1$s`/`%2$d` positional argument present exactly once, positions
      preserved (translators reorder words — positional indices make that safe; a missing arg
      crashes at runtime). Diff arg counts per string between base and translated files.
    - Apostrophes escaped (`\'`), no raw `&`, placeholders like `%%` intact.
12. **RTL smoke test if AR/HE targeted**: verify `android:supportsRtl="true"` in the manifest,
    run the app under `ar-XB`, and walk the top 5 screens checking mirrored layout, no
    `left/right` attributes where `start/end` belong, and icon mirroring for directional glyphs.
13. Re-run lint and confirm zero `MissingTranslation` errors for declared locales (or explicitly
    mark intentionally-untranslated strings `translatable="false"`).

### Stage 5 — Per-Store Locale Mechanics

14. **Play**: add each language under Store presence → Main store listing (Play also supports
    **custom store listings** per country/region — use one when a locale needs different
    *positioning*, not just different language, e.g. a different lead benefit for ID). Localized
    screenshots upload per language. Details in `04-aso/stores/google-play.md`.
15. **AppGallery**: add languages in AppGallery Connect listing-information locale fields; each
    locale carries its own name, intro, and screenshots. **RuStore**: complete the Russian locale
    fields as primary; see `04-aso/stores/huawei-appgallery.md` and `04-aso/stores/rustore.md`
    for field-by-field mechanics and limits.

### Stage 6 — QA Checklist (gate before release; feeds `07-checklists/release-readiness.md`)

16. Run for every Tier 2+ locale:
    - [ ] No truncation/clipping at **200% font scale** in the **longest target locale**
          (test the worst case: usually DE or RU) on a small-screen device profile.
    - [ ] No hardcoded strings left: lint reports zero `HardcodedText`.
    - [ ] Zero `MissingTranslation` lint errors for all declared locales.
    - [ ] Plurals and format-args integrity checks (step 11) passed for every file.
    - [ ] RTL smoke test passed if AR/HE shipped.
    - [ ] Store listing fields render correctly in each store console preview (no mid-word
          truncation of title/short description at store limits).

### Stage 7 — Measurement

17. After 28 days per locale, record in `docs/factory/audits/ASO_AUDIT.md` (the only ASO_AUDIT
    path — `01-core/CONVENTIONS.md`): per-locale
    store-listing conversion rate vs pre-localization baseline (Play Console, by country),
    install volume delta, and any review-language sentiment shift. Promote locales that convert
    (Tier 1 → 2 → 3) and freeze locales showing no lift after one keyword-map refresh cycle.

## Outputs

| Artifact | Path |
|---|---|
| Locale prioritization table + tier assignments | `docs/factory/audits/ASO_AUDIT.md` (localization section) |
| Per-locale keyword maps | `docs/factory/reports/KEYWORD_MAP.md` (per-locale sections) |
| Localized listing copy | `docs/factory/reports/STORE_LISTING_<LOCALE>.md` from `09-templates/store-listing.md` |
| Localized screenshot sets | `docs/factory/assets/store/<store>/<locale>/` |
| Translated string resources | Target app repo `res/values-<locale>/strings.xml` |
| CHANGELOG entry | `docs/factory/CHANGELOG.md` |

## Acceptance Criteria

- [ ] Locale decisions scored on install geo × store opportunity vs cost; RU at Tier 1+ wherever
      the app ships on RuStore; every locale has an explicit tier.
- [ ] Keywords researched natively per locale via suggest harvesting — zero translated keyword
      slots; all volume/difficulty values labeled `(est.)`.
- [ ] Title and short description native-reviewed, or locale flagged `pending native review` with
      the conservative-title fallback applied.
- [ ] Tier 2+ locales: pseudo-locale (`en-XA`) pass done before translation; plurals and
      format-args verified; lint clean for `MissingTranslation` and `HardcodedText`.
- [ ] RTL smoke test passed for any AR/HE target.
- [ ] Stage 6 QA checklist fully passed per locale before release sign-off.
- [ ] Per-locale conversion deltas scheduled for the +28-day review and recorded in
      `docs/factory/audits/ASO_AUDIT.md`.
