# obsidian-plugin-lint

A [Claude Code](https://claude.com/claude-code) / Agent **skill** that runs a static-analysis lint pass over an Obsidian plugin before release.

It covers everything the Obsidian community **submission bot** flags, plus the official ESLint rules and CSS best practices — each rule keyed to the exact warning message so you can paste a bot warning and get the fix:

- **Official ESLint rules** (`obsidianmd/eslint-plugin`) — inline styles, `innerHTML`, default hotkeys, `this.app`, `normalizePath()`, sentence case
- **CSS best practices** — CSS variables, `!important`, full 6-digit hex, duplicate properties, `:has()` performance
- **Popout-window safety** — `activeDocument` vs `window.` timers vs `.instanceOf()`, with the exact bot wording for each
- **TypeScript type-safety** — `loadData()` casts, `strictBindCallApply`, `no-misused-promises`, unnecessary assertions, regex pitfalls
- **manifest.json / dependencies / release workflow** — `minAppVersion` API gating, `pnpm audit` overrides, build-provenance attestation

## Install

```bash
# Claude Code (user-level skills)
git clone https://github.com/tcyeee/obsidian-plugin-lint ~/.claude/skills/obsidian-plugin-lint
```

Or copy the folder into any agent's skills directory — the skill is the single `SKILL.md`.

## Use

Invoke it explicitly:

```
/obsidian-plugin-lint
```

…or just describe the task. The skill auto-triggers on phrases like `lint`, `检查代码`, `check styles`, or when you paste an Obsidian submission-bot warning such as *"Avoid !important"*, *"Use the full 6-digit hex format"*, or *"Unexpected duplicate"*.

## Report format

```
[FILE:LINE]  <full offending line>
  Rule:  <rule name>
  Fix:   <what to change>
```

Intentional exceptions are marked `[KEPT: reason]`.

## License

[MIT](./LICENSE)
