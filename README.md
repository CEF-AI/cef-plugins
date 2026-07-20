# cef-plugins

Claude Code plugin marketplace for **CEF agent development**.

## Install

```
/plugin marketplace add CEF-AI/cef-plugins
/plugin install cef@cef
```

Projects scaffolded with `cef init` auto-wire this plugin via
`.claude/settings.json` — trust the folder and Claude Code installs and keeps
it updated.

## Plugins

### `cef`

Author, test, build, push, and deploy CEF agents. Skills:

- **`cef:develop`** — author, extend, and debug agents (engagements, cubbies,
  migrations, widgets, tests). Loads automatically when you work on agent code.
- **`cef:deploy`** — the `build → push → publish → deploy` workflow; confirms
  before outward-facing steps and guides the human-only ROC / credentials
  actions.

Depth lives in each skill's `references/`.
