# ClaudeVerse

A [Claude Code skill](https://code.claude.com/docs/en/skills) for the Verse programming language and the UEFN Scene Graph, including the Verse/UnrealEngine/Fortnite APIs.

Claude loads `SKILL.md` on its own when a task touches Verse or UEFN, and reads the deeper reference files as needed:

- `reference/language.md`: types, literals, variables, operators, functions, classes, modules, advanced types
- `reference/effects-failure-concurrency.md`: effect specifiers, failure, STM rollback, concurrency, events, live variables
- `reference/scene-graph.md`: entities, components, lifecycle, queries, scene events, transforms
- `reference/apis.md`: fort_character, damage/health, playspaces, teams, inventories, devices, UI widgets
- `reference/gotchas.md`: common mistakes and the right idiom
- `reference/patterns.md`: component and system patterns

## Install

For all your projects:

```bash
git clone https://github.com/magnusenebakk-epic/claudeverse.git ~/.claude/skills/claudeverse
```

For a single project, so everyone who clones it gets the skill:

```bash
git clone https://github.com/magnusenebakk-epic/claudeverse.git <project>/.claude/skills/claudeverse
```

The skill triggers on its own when you work on Verse code. Invoke it explicitly with `/claudeverse`. Update with `git pull`.

## Notes

- Written for the current toolchain, where functions default to `<no_rollback>` and every `<decides>` function needs an explicit `<computes>` or `<transacts>`. Update the skill when that changes.
- Some helpers in `patterns.md` (`RecursiveSync`, `GetFirstDescendantComponent`, ...) are utils-file extension methods, not engine APIs. Their definitions are included so any project can adopt them.
- The generated `*.digest.verse` files for your build are the source of truth for exact API signatures.
