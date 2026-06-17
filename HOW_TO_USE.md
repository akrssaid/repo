# How To Use With Claude Code

## 1. Copy Into A Target Project

Put this folder inside the Android project you want to update. Recommended folder name:

```text
codex-android-update-factory
```

Expected layout:

```text
MyAndroidApp/
├── settings.gradle.kts
├── app/
└── codex-android-update-factory/
```

## 2. Start Claude Code From The Target Root

Open Claude Code in `MyAndroidApp/`, not inside the factory folder.

First message:

```text
Read codex-android-update-factory/01-core/CLAUDE_MASTER.md and adopt it as your operating
instructions for this Android project.
```

For a brand-new engagement, continue with:

```text
Execute codex-android-update-factory/01-core/prompts/01-discovery.md against this repository.
```

## 3. Keep Project State In The Target Repo

Claude Code writes all app-specific output here:

```text
docs/factory/
├── PROJECT_STATE.md
├── CHANGELOG.md
├── audits/
├── handovers/
├── reports/
└── assets/
```

Do not put app-specific discoveries, screenshots, handovers, or release notes inside
`codex-android-update-factory/`.

## 4. Resume Later

In later Claude Code sessions, start from the Android project root and paste:

```text
Read codex-android-update-factory/01-core/CLAUDE_MASTER.md, then bootstrap from
docs/factory/PROJECT_STATE.md and docs/factory/CHANGELOG.md. Continue with the next action.
```

## 5. Run A Specific Phase

After P1 and P2 are complete, you can ask for a specific phase:

```text
Read codex-android-update-factory/01-core/CLAUDE_MASTER.md and execute
codex-android-update-factory/01-core/prompts/03-modernization-audit.md.
```

The master instructions decide whether prerequisites are satisfied and where outputs belong.

