# Codex Android Update Factory

Portable Claude Code workflow kit for updating existing Android projects.

This repository is not an Android app. It is a prompt, checklist, template, and handover system
that you copy into a target Android project when you want Claude Code to run a structured update:
discovery, modernization, redesign, ASO, verification, and release preparation.

## Intended Use

Copy this folder into the root of the Android project you want to update:

```text
<target-android-project>/
├── settings.gradle(.kts)
├── app/
├── codex-android-update-factory/
└── docs/factory/
```

Then open Claude Code in the target project root and paste:

```text
Read codex-android-update-factory/01-core/CLAUDE_MASTER.md and adopt it as your operating
instructions for this Android project. Start with P1 Discovery by executing
codex-android-update-factory/01-core/prompts/01-discovery.md.
```

Claude Code should write project-specific state and reports only under `docs/factory/` in the
target Android project. The copied `codex-android-update-factory/` folder is treated as read-only
workflow material.

## What Is Inside

| Path | Purpose |
|---|---|
| `01-core/` | Master instructions, lifecycle, conventions, and executable phase prompts |
| `02-engineering/` | Android audit, dependency, SDK, manifest, build, testing, and migration procedures |
| `03-design/` | Design audit, screenshots, Material 3, icon, splash, motion, and accessibility procedures |
| `04-aso/` | Store and ASO workflows for Google Play, Huawei AppGallery, and RuStore |
| `05-handover/` | Handover schemas between audit, design, implementation, ASO, and release phases |
| `06-playbooks/` | App-category playbooks |
| `07-checklists/` | Phase gates and validation checklists |
| `08-knowledge/` | Android, design, ASO, store-policy, monetization, and performance references |
| `09-templates/` | Report templates instantiated into the target repo's `docs/factory/` |
| `10-releases/` | Release workflow and app versioning policy |

## Lifecycle

Run the workflow in phases:

1. P1 Discovery
2. P2 Project Memory
3. P3 Engineering Audit & Roadmap
4. P4 Modernization
5. P5 Design Audit & Handover
6. P6 Redesign Implementation
7. P7 ASO & Store Assets
8. P8 Verification & Release

The quick reference is `QUICK_REFERENCE.md`; the canonical lifecycle is
`01-core/PROJECT_LIFECYCLE.md`.

