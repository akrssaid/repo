# Template — Screen Inventory

This template produces the authoritative catalogue of every screen in a target app, with entry
points, states, baseline screenshots, and a flow map. It is the shared coordinate system for the
whole design pipeline: `DESIGN_AUDIT.md` findings, the Design Handover, redesign implementation,
and store screenshot generation all reference screens by the SCR-xxx IDs minted here. Completeness
is the contract — the Coverage Statement must reconcile every discovery source.

## Usage

- **Instantiated by:** the design auditor agent during `01-core/prompts/05-design-audit.md`,
  executing the method in `03-design/02-screen-inventory.md` (capture procedure in
  `03-design/03-screenshot-capture.md`).
- **Phase:** P5 — Design Audit & Handover (created before `DESIGN_AUDIT.md`).
- **Instance path:** `docs/factory/audits/SCREEN_INVENTORY.md` in the TARGET app repo.
- **Living document:** the States and Capture Gaps columns/sections are updated as missing
  captures are filled during P5–P6; SCR-xxx IDs are never renumbered or reused.
- **Procedure:** Instantiation Doctrine in `09-templates/README.md` — copy body, fill all tokens,
  delete guidance comments and example rows, verify zero `{{` remain.

## Field Guide

| Token | Meaning | Example |
|---|---|---|
| `{{APP_NAME}}` / `{{PACKAGE_NAME}}` | App name / applicationId | Solar Camera Pro / com.solarcamera.pro |
| `{{INVENTORY_DATE}}` | Date inventory completed, ISO 8601 | 2026-06-10 |
| `{{COMMIT_SHA}}` | Git SHA screenshots were captured from | 9f3c1a7e… |
| `{{DEVICE_PROFILE}}` | Emulator/device used for baseline captures | Pixel 8, API 36 emulator, 1080×2400 |
| `{{FACTORY_VERSION}}` | Factory version from `VERSION.md` | 1.2.0 |
| `{{MANIFEST_ACTIVITY_COUNT}}` | Activities declared in merged manifest | 6 |
| `{{NAV_DESTINATION_COUNT}}` | Destinations across nav graphs / Compose NavHosts | 11 |
| `{{DYNAMIC_SCREEN_COUNT}}` | Screens found only by runtime exploration (dialogs-as-screens, ViewPager pages, etc.) | 3 |
| `{{TABLET_PROFILE}}` | Tablet/large-window device used for adaptive captures, or "None — phone-only app" | Pixel Tablet, API 36 emulator, 2560×1600 |
| `{{TOTAL_SCREEN_COUNT}}` | Total unique SCR-xxx rows in the inventory | 14 |
| `{{RECONCILIATION_NOTE}}` | Sentence(s) explaining how source counts map to the total (overlaps, exclusions) | "6 activities host 11 nav destinations; 3 destinations are duplicates of activity roots…" |
| `{{FLOW_MAP_ASCII}}` | ASCII diagram of primary user flows using SCR-xxx IDs | (diagram) |

---

<!-- ──────────────── TEMPLATE BODY — copy everything below this line ──────────────── -->

# Screen Inventory — {{APP_NAME}}

| Field | Value |
|---|---|
| App | {{APP_NAME}} (`{{PACKAGE_NAME}}`) |
| Inventory date | {{INVENTORY_DATE}} |
| Commit captured | `{{COMMIT_SHA}}` |
| Capture device (phone) | {{DEVICE_PROFILE}} |
| Capture device (tablet/large) | {{TABLET_PROFILE}} |
| Factory version | {{FACTORY_VERSION}} |

## Coverage Statement

<!-- This section is the completeness proof. Enumerate every discovery source and reconcile the
     arithmetic — the reader must be able to verify that no screen escaped. Sources, in order:
     1) merged-manifest activities (aapt2 dump or manifest read), 2) navigation destinations
     (XML nav graphs + Compose NavHost routes), 3) runtime exploration (screens reachable only
     dynamically: full-screen dialogs, ViewPager/Pager pages, settings subscreens).
     The reconciliation note must explain overlaps (an activity that is also a nav host) and
     deliberate exclusions (e.g. trampoline activities with no UI). Totals MUST reconcile. -->

| Source | Count |
|---|---|
| Manifest activities | {{MANIFEST_ACTIVITY_COUNT}} |
| Navigation destinations | {{NAV_DESTINATION_COUNT}} |
| Dynamically discovered | {{DYNAMIC_SCREEN_COUNT}} |
| **Unique screens (inventory total)** | **{{TOTAL_SCREEN_COUNT}}** |

**Reconciliation:** {{RECONCILIATION_NOTE}}

## Inventory

<!-- One row per unique screen, per the method in 03-design/02-screen-inventory.md Step 3
     (column set reconciles with that method — do not invent a different schema). Rules:
     - ID: SCR-001 ascending in rough flow order (launcher first). IDs are permanent.
     - Class / route: FQCN or Compose route — the code anchor for implementers. (This replaces
       the old "Module" column; module is part of the class path when it matters.)
     - Entry points: how a user reaches it (launcher, nav action from SCR-xxx, deep link,
       notification, share intent). "unknown" is not allowed — verify or mark unreachable.
     - Toolkit: XML / Compose / Mixed.
     - Priority: H = core-task path / first session, M = regular secondary feature, L = rare
       (settings depths, legal pages).
     - Monetization: banner / interstitial-entry / native / paywall / upsell, or "—" if none.
     - States (Default/Empty/Loading/Error/Offline/Dark): Y/N per state captured as a screenshot.
       "n/a" where a state cannot exist (no error on a static About screen) — n/a counts as
       covered. default + dark ALWAYS; offline whenever the screen does network I/O (offline = no
       connectivity, distinct from error = request failed while connected).
     - Adaptive / orient.: phone-only / responsive / tablet+, plus portrait / landscape / both;
       tablet+/landscape/both screens drive the extra captures and 03-design/11-adaptive-layout.md.
     - Baseline path: this screen's captures, named
       docs/factory/assets/screenshots/baseline/SCR-<id>-<state>-<theme>.png
       (e.g. SCR-001-default-light.png, SCR-001-default-dark.png — the theme segment is mandatory). -->

| ID | Name | Class / route | Entry points | Purpose | Toolkit | Priority | Monetization | Default | Empty | Loading | Error | Offline | Dark | Adaptive / orient. | Baseline screenshots |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| SCR-001 | Home / Gallery | `HomeActivity` (`:app`) | Launcher; back from all flows | Browse captured photos, entry to camera | XML | H | banner | Y | Y | Y | N | N | N | responsive / both | `docs/factory/assets/screenshots/baseline/SCR-001-<state>-<theme>.png` | <!-- (example — replace) -->

## Flow Map

<!-- ASCII diagram of the PRIMARY user flows only (3–5 flows max), using SCR-xxx IDs with short
     labels. Arrows show navigation direction; annotate forks. Keep it readable in a terminal.
     Example shape:
       SCR-001 Home ──capture──> SCR-002 Camera ──save──> SCR-003 Editor ──share──> [share sheet]
            │
            └─settings──> SCR-009 Settings ──> SCR-010 About
     Secondary/rare paths stay out — the table's entry-points column already records them. -->

```text
{{FLOW_MAP_ASCII}}
```

## Capture Gaps

<!-- Every N in the States columns gets a row here: why it is missing and how/when it will be
     captured. This log is worked down during P5–P6; a gap that cannot be captured (e.g. error
     state requires a server fault) gets resolution "Simulated via <method>" or "Accepted —
     documented in DESIGN_AUDIT.md". Empty is valid only when every state cell is Y or n/a —
     then write "None — full coverage as of {{INVENTORY_DATE}}". -->

| Screen | Missing state | Reason | Plan | Status |
|---|---|---|---|---|
| SCR-001 | Error | Requires storage-permission denial mid-session | Re-capture with `adb shell pm revoke {{PACKAGE_NAME}} android.permission.READ_MEDIA_IMAGES` | In Progress | <!-- (example — replace) -->

## Maintenance Rules

- New screens added by the redesign get the next free SCR-xxx ID; never renumber existing rows.
- Re-captures after the redesign go to `docs/factory/assets/screenshots/redesigned/` (never
  `redesign/`) — the `baseline/` directory is immutable evidence for `DESIGN_AUDIT.md`.
- All captures, baseline and redesigned, follow the `SCR-<id>-<state>-<theme>.png` naming
  convention (theme segment mandatory; dark captured for every inventoried screen).
- Any change to this inventory is logged in `docs/factory/CHANGELOG.md`.
