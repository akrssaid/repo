# UX Writing Essentials

The factory's reference for **interface copy** — the words on buttons, in empty states, in error
messages, in permission primers (D13). UX writing is a design discipline: the same redesign rigor
applied to layout applies to language. This doc is doctrine and patterns; it is consumed by the
design audit (copy is scored under heuristics H5/H6 in `03-design/01-design-audit.md`), the redesign
proposal (which may propose copy changes), and prompt 09 (which implements approved copy via COPY
tickets — §7). Voice/tone interplay with the visual system is in
`08-knowledge/design/material3-essentials.md`; broader UX doctrine in
`08-knowledge/design/mobile-ux-principles.md`.

**Last reviewed: 2026-06-10.** Copy doctrine is stable; only platform-specific string conventions
and localization-pipeline specifics drift — **verify against current official docs** (Material
"Writing" guidance, Android string-resource docs) where a convention is load-bearing.

---

## 1. The universal microcopy structure: what / why / next

Almost every functional message — error, empty state, permission primer, blocking dialog — answers
three questions, in this order:

1. **What happened** (or what is true): the situation, in the user's terms.
2. **Why** (when it helps): the cause or benefit, only if it changes what the user does.
3. **Next step**: the one action that moves them forward — a button, a link, a clear instruction.

Drop the "why" when it's obvious or unhelpful; never drop "what" or "next". A message with no next
step is a dead end.

### Error messages

Blame the situation, not the user; never show a raw exception or a bare "Something went wrong."

| Bad | Good |
|---|---|
| "Error 0x80." | "Couldn't open this file. It may be moved or deleted. **Choose another file**." |
| "Network error." | "No connection, so the download is paused. It'll resume automatically when you're back online. **Retry now**." |
| "Permission denied." | "Photos access is off, so this folder can't be scanned. **Turn on in Settings**." |

### Empty states

An empty state **teaches** (see `mobile-ux-principles.md` §3): what belongs here, why it's useful,
and the one action that fills it.

| Bad | Good |
|---|---|
| "No items." | "No scans yet. Scanned documents appear here as searchable PDFs. **Scan a document**." |
| "Nothing found." | "No files match \"report\". Try a shorter word or **clear the search**." |

### Permission primers

The primer is the *in-app* line shown **before** the system permission dialog (the priming pattern,
`mobile-ux-principles.md` §7). It states the benefit in the user's terms, then the system dialog
fires. On denial, degrade gracefully — never re-loop.

> "To recover deleted photos, the app needs access to your storage. Nothing is uploaded — recovery
> runs on your device." → then the system dialog.

---

## 2. Button & action labels

- **Verb the user's intent.** Labels describe what *happens when tapped*, from the user's side:
  `Scan`, `Delete`, `Restore`, `Allow`, `Turn on`. Not vague (`OK`, `Submit`, `Done` where a real
  verb fits) and not system-voiced (`Enable feature`).
- **Match the dialog's question.** A "Delete 3 files?" dialog gets **Delete** / **Cancel**, never
  **Yes** / **No** — the verb-paired choice is faster and safer to read.
- **One primary action per surface.** The confirming verb is the emphasized button; the escape is
  the quieter one. Never two competing filled buttons.
- **Destructive verbs are explicit.** `Delete permanently`, not `Confirm`; the word carries the
  weight so the button color isn't the only warning.
- **No dead labels.** Avoid `Got it` / `OK` on anything that should offer a real next step.

---

## 3. Mechanics: case, length, person, tone consistency

- **Sentence case** for everything: buttons, titles, menu items, labels (`Turn on notifications`,
  not `Turn On Notifications` and not `Turn on Notifications`). Title Case is a dated, harder-to-scan
  convention on Android.
- **Second person, active voice.** Address the user as "you"; the app is mostly invisible ("Choose a
  file", not "The user should choose a file"; "Couldn't save", not "The system was unable to save").
- **Numerals as digits** in UI (`5 files`, not `five files`); respect locale formatting for numbers,
  dates, currency.
- **Be consistent**: one term per concept across the whole app (`Folder` everywhere, never
  `Folder`/`Directory`/`Location` for the same thing). Build a tiny per-app term glossary during the
  redesign and hold to it.
- **No jargon, no internal names.** Users don't know what a "sync adapter" or "cache partition" is.

---

## 4. Text-length budgets & localization expansion

Copy is written in the source language but must survive translation. Translated text **expands** —
budget for it or the redesign breaks in market.

### Expansion table (approximate; verify per target locale)

| Target language | Typical expansion vs. English |
|---|---|
| German | **+30–40%** (long compound nouns; the worst common offender) |
| Russian | **+30–40%** |
| French | +15–25% |
| Spanish / Portuguese / Italian | +15–25% |
| Polish, Finnish, Hungarian | +30–40% (long words / agglutination) |
| Arabic, Hebrew | similar length but **RTL** — layout mirrors (start/end, never left/right) |
| Chinese, Japanese, Korean | often **shorter**, but glyphs are taller/denser |

Practical budgets (source English): keep **button labels ≤ ~12 characters** where possible, **titles
≤ ~30**, and never design a control that only fits if the word stays English. Allow text to wrap or
the container to grow; choose ellipsis policy deliberately. The pseudo-locale sweep (`en-XA` accented,
`ar-XB` pseudo-RTL) catches overflow before translators are involved — procedure and the
`HardcodedText`-lint rule are in `08-knowledge/android/common-pitfalls.md` (RL4) and
`04-aso/workflows/localization.md`. Every user-facing string lives in resources; no hardcoded text.

---

## 5. Deriving tone from brand adjectives

Tone is not invented per screen — it's **derived** from the brand's adjectives (captured in the
DESIGN handover's brand section). Pick 3–4 adjectives, translate each into a writing rule, then write
every string against the rules.

| Brand adjective | Becomes the writing rule |
|---|---|
| Trustworthy | Plain, specific, no hype; honest about errors ("Couldn't upload — your file is safe and still here") |
| Efficient | Short; lead with the action; no filler greetings |
| Friendly | Contractions, warmth in empty states — but never cute in errors or on destructive actions |
| Professional | No slang, no exclamation marks, restrained |

Tone **flexes by moment**: warmer in empty states and success, plainer and calmer in errors,
neutral and unmistakable on destructive confirmations. A "friendly" brand still writes a delete
warning soberly. Record the adjectives→rules mapping once per app so every writer (and prompt 09)
applies the same voice.

---

## 6. Copy review heuristics (what audits flag)

During the design audit, copy is evaluated under heuristics **H5/H6** (`03-design/01-design-audit.md`)
and logged as DES findings. Common flags:

- Raw exceptions / error codes shown to users; bare "Something went wrong."
- Empty states that don't teach (bare "No items").
- Permission requested with no in-app primer, or primer that doesn't state the benefit.
- `Yes/No` (or `OK`) where verb-paired labels belong; vague button labels.
- Title Case, inconsistent terminology, internal jargon.
- Strings that will overflow in DE/RU; hardcoded (non-resource) text.

Each becomes a DES finding with severity per `01-core/CONVENTIONS.md` scales and, if a wording change
is warranted, a proposed **Copy Change** that flows into the COPY-ticket pipeline below.

---

## 7. The COPY-NNN ticket flow (how copy changes become allowed in P6)

The old blanket **string-freeze is abolished** (D13): un-ticketed string changes remain banned, but
copy *can* change in P6 when it goes through the COPY pipeline. The end-to-end flow:

1. **Audit (P5).** The design audit surfaces copy problems under H5/H6 as DES findings with evidence
   (`03-design/01-design-audit.md`).
2. **Handover declares constraints (prompt 06).** The DESIGN handover's *Localization & Content
   Constraints* section declares which strings are **frozen** (legal, brand-locked, store-claim-tied)
   vs **redesignable** (D13/D15).
3. **Proposal proposes (prompt 07).** The REDESIGN proposal may propose specific **Copy Change**
   items — old string → new string, with rationale — for redesignable strings only.
4. **Handover emits tickets (prompt 08).** Each accepted Copy Change becomes a **`COPY-NNN`** ticket
   (the COPY ID stream, `PREFIX-NNN` zero-padded, registered in `01-core/CONVENTIONS.md` D2),
   carrying: the string key/resource, old text, new text, screens affected, and locale notes.
5. **Prompt 09 implements.** Implementation edits **only** the strings named by open COPY tickets,
   in the resource files, and updates the ticket status. A string change with no COPY ticket is still
   a violation; a frozen string is never touched.

This is the *only* legal path for copy to change during a redesign. It keeps localization stable
(translators re-translate exactly the ticketed deltas) while removing the blunt freeze that used to
block obviously-needed fixes.

---

## 8. Cross-references

- States that microcopy lives in (empty/error/offline): `08-knowledge/design/mobile-ux-principles.md` §3
- Copy heuristics in the audit rubric (H5/H6): `03-design/01-design-audit.md`
- Localization workflow & pseudo-locale sweep: `04-aso/workflows/localization.md`
- Overflow / hardcoded-string pitfalls: `08-knowledge/android/common-pitfalls.md` (RL4)
- COPY ID registry & scales: `01-core/CONVENTIONS.md`
- The handover/proposal/ticket chain: prompts 06 → 07 → 08 → 09
