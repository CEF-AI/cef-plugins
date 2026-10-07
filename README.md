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

Author, test, build, push, deploy, and evaluate CEF agents. Skills:

- **`cef:develop`** — author, extend, and debug agents (engagements, cubbies,
  migrations, widgets, tests). Loads automatically when you work on agent code.
- **`cef:deploy`** — the `build → push → publish → deploy` workflow; confirms
  before outward-facing steps and guides the human-only ROC / credentials
  actions.
- **`cef:evaluate`** — `cef eval`: keep a workflow's dataset of cases in the
  repo, sync it with the agent service's bucket, run it as an experiment
  against a workflow version, and compare experiments with a baseline.

Depth lives in each skill's `references/`.
