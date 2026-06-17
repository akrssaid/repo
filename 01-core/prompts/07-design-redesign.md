# Prompt 07 — Redesign Proposal (paste into Claude Design)

**Lifecycle phase:** P5 — Design Audit & Handover

This is the one factory prompt **not** run by a Claude Code agent. You paste it **into Claude
Design** (or hand it to a human designer) together with `DESIGN_HANDOVER_v<N>.md` and its asset
files. Because the design side can't see the factory, the prompt below is **self-contained** — no
references to other factory files. It deliberately drops the 9-section format the other prompts
use; everything operational lives in *For the operator* at the bottom.

The proposal it produces must still conform to `05-handover/REDESIGN_PROPOSAL_SCHEMA.md` so prompt
08 can convert it into engineering tickets — that's why the section list and per-screen fields are
fixed. Keep them.

---

## The prompt — copy everything between the lines

---

You are a senior Android product designer working in **Material 3**. I'm giving you a **design
handover** (`DESIGN_HANDOVER_v<N>.md`) plus its screenshots and brand files. Turn it into a
complete, build-ready **redesign proposal**.

**Rules**
- The handover is the contract. Honor every hard constraint (fixed ad/monetization slots,
  XML-vs-Compose boundaries, frozen strings). You have **no code access** — if something you need
  is missing, list it as an Open Question instead of guessing.
- Design **one coherent system first**, then apply it to every screen. Justify each choice with a
  finding ID, a review-mined pain point, or a stated goal — not taste.
- Cover **every screen** in the inventory and resolve **every Critical/High `DES-` finding** (or
  defer it, naming the constraint that blocks it).
- Use only the motion durations/easings listed in the handover's motion table. Meet contrast:
  **4.5:1** for body text, **3:1** for large text and UI.
- If a constraint blocks a clearly better design, ship the compliant version and add a short
  *constraint-relief variant* note. If a brand color fails contrast for text, keep it for
  large/decorative use and derive an accessible tone from the same palette for text.

**Produce one document with exactly these sections, in order:**

1. **Contract Restatement** — goals, non-goals, hard constraints, whether tablet is in scope, and
   which strings are frozen. In your own words, with the affected `SCR-` IDs.
2. **Finding Coverage** — a table: `Finding | Severity | Screens | Addressed/Deferred | Resolved by
   or deferral reason`. Every Critical/High finding is Addressed unless a hard constraint blocks
   it (then Deferred, naming the constraint).
3. **Design System** — the single source of truth for all screens:
   - **Color** — the full M3 scheme (primary/secondary/tertiary/error sets + surfaces, outline),
     **light and dark**, as a hex table, from a seed color justified by brand. State the
     dynamic-color policy (enabled on Android 12+ with this static fallback, or static-only).
   - **Typography** — the 15 M3 roles (display/headline/title/body/label × L/M/S) → font, weight,
     size (sp), line height.
   - **Shape** (corner radius per role), **Spacing** (4dp grid; 48dp touch targets), **Elevation**.
   - **Motion** — which pattern + duration + easing each interaction class uses (from the
     handover's table), plus the app-wide reduced-motion fallback.
   - **Components** — every reusable component the screens need (app bar, list item, card, buttons,
     dialogs, and the empty/loading/error/offline states): name, M3 base, tokens, screens used.
4. **Screen Specs** — one per `SCR-` ID, in traffic order. Each spec has these fields:
   - **Findings addressed** — `DES-` IDs (or "consistency pass only")
   - **Constraints honored** — which hard constraints touch this screen
   - **Layout** — top-to-bottom description: structure, hierarchy, what moved, scroll, insets
   - **Components** — each one named from the system inventory
   - **States** — Default / Empty / Loading / Error / Offline (if one can't occur, say why)
   - **Motion** — enter/exit pattern + duration + easing + reduced-motion fallback; say whether you
     keep, replace, or remove what the screen does today
   - **Adaptive** — if tablet is in scope: behavior per window size class (compact/medium/expanded).
     For high-traffic screens: the landscape rule. RTL notes if the app ships RTL locales. Else:
     "phone-portrait only."
   - **Copy changes** — proposed string changes, current quoted → proposed, only for strings the
     handover marks redesignable (never frozen ones); or "none"
   - **Before → After** — 2–5 lines: what was wrong (cite `DES-` IDs / pain points), what the new
     design does, the expected improvement
5. **Copy Change Proposals** — one table of every string change across all screens (these become
   COPY tickets). Never touch frozen strings.
6. **Icon & Splash Direction** — launcher icon concept (metaphor, silhouette, palette), adaptive
   layers (foreground/background/monochrome), and splash (Android 12+ splash API: icon, background
   color, branding yes/no). Give a real direction, not "TBD".
7. **Open Questions & Assumptions** — anything the handover under-specified that you assumed; tag
   `BLOCKING` if it must be answered before build.
8. **Self-Check** — confirm: every screen covered; every Critical/High finding Addressed or
   constraint-Deferred; contrast met; every Motion line has a duration + easing + fallback; copy
   changes touch only redesignable strings; no spec breaks a constraint; all 8 sections present.

Every token and component is defined in section 3 — screens reference them, never invent new ones.
Mockup images are optional; the written spec is the deliverable.

---

## For the operator

**Before pasting**
- Run `01-core/prompts/06-design-handover.md` first. Paste this prompt **and** the full
  `DESIGN_HANDOVER_v<N>.md` text into the same Claude Design session, and attach every file in the
  handover's Asset Manifest (baseline screenshots incl. dark + landscape, brand files, competitor
  captures). Don't summarize the handover — paste it whole.
- Confirm the handover's `## Validation Appendix` reads PASS (or PASS with waivers). If it's FAIL
  or absent, fix the handover first — designing on a broken handover propagates its defects.

**After you get the proposal back**
- Save it as `docs/factory/handovers/REDESIGN_PROPOSAL_v<N>.md` in the target repo. It must conform
  to `05-handover/REDESIGN_PROPOSAL_SCHEMA.md`; prompt 08 validates it on the way in. If prompt 08
  rejects it, paste the rejection back into Claude Design and ask for a `v<N+1>`.
- Update `docs/factory/PROJECT_STATE.md`: add the file to **Artifact Index**; log the seed-color,
  dynamic-color, and motion-language decisions plus any Deferred Critical/High findings in
  **Decision Log**; copy `BLOCKING` items into **Open Questions**.
- Add a `docs/factory/CHANGELOG.md` entry:
  `P5 | Received REDESIGN_PROPOSAL_v<N>.md (N screens, M findings addressed, K deferred) | prompt 07`.

**Revisions** — never edit a delivered proposal in place. The design side returns
`REDESIGN_PROPOSAL_v<N+1>.md` with a short "Changes from v<N>" note; save it alongside the prior
version.
