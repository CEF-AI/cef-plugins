# Project anatomy & runtime model

A CEF agent is authored as a TypeScript project, compiled by `cef build` into a
bundle + manifest, and executed by the Cere Agent Runtime (AR ADR-001). This
reference maps the files you'll touch to what the runtime and orchestrator
actually consume.

## What a project contains

```
my-agent/
  cef.config.ts        # declarative agent manifest source (defineAgent)
  src/agent.ts         # default-exported engagement class w/ decorators
  migrations/          # per-cubby SQL migration dirs (one dir per cubby alias)
    history/001-init.sql
  widgets/             # optional: built, self-contained static web UIs
  test/*.test.ts       # vitest + @cef-ai/testing simulator tests
  package.json         # depends on @cef-ai/agent-sdk (+ -D @cef-ai/cli)
  .cef/generated.d.ts  # emitted by `cef typegen` — KnownEventTypes autocompletion
  cef.lock.json        # typegen snapshot; VCS diffs surface drift
  dist/<agentId>/      # `cef build` output (bundle.js + manifest.json)
```

- **`cef.config.ts`** — the single source of declarative config. Exports
  `defineAgent({...})` as default. Declares identity, engagements, cubbies,
  widgets, models, settings, params. `cef build` turns it into the manifest.
- **`src/agent.ts`** — the code. A default-exported class decorated with
  `@Engagement`, `@OnEvent`, `@OnStart`, `@OnClose`. This is the *entry* the
  config points at (`entry` or `engagements[].entry`).
- **`migrations/<alias>/*.sql`** — SQL schema for a cubby. A `CubbyDecl` in the
  config (`{ alias, migrations: "./migrations/history" }`) points at the dir;
  `ctx.cubby("history")` at runtime reads/writes that SQLite-backed store.
- **`widgets/`** — optional built static UIs (entry HTML + sibling assets using
  `./relative` refs). `cef push` uploads each `WidgetDecl.dir` to DDC as ONE
  content-addressed directory DAG; the manifest carries `{ bucketId, cid, entry }`
  and consumers resolve at `<gateway>/<bucketId>/<cid>/<entry>`. The `dir` must
  already be built before push — the CLI copies, it does not bundle widgets.

### Minimal example

```ts
// cef.config.ts
import { defineAgent } from "@cef-ai/agent-sdk/config";

export default defineAgent({
  id: "echo",
  version: "0.1.0",
  entry: "./src/agent.ts",
  cubbies: [{ alias: "history", migrations: "./migrations/history" }],
});
```

```ts
// src/agent.ts
import { Engagement, OnEvent, OnStart, type Context, type Event } from "@cef-ai/agent-sdk";

@Engagement({ id: "default", goal: "echo back any user message" })
export default class Echo {
  @OnStart async onStart() { console.info("started"); }

  @OnEvent("user_message")
  async onMessage(event: Event<{ text: string }>, ctx: Context) {
    await ctx.cubby("history").exec(
      "INSERT INTO messages(id, text, ts) VALUES (?, ?, ?)",
      [Date.now(), event.payload.text, event.ts],
    );
    await ctx.publish("ack", { for: event.id });
  }
}
```

## What `defineAgent` accepts

`defineAgent<T extends AgentConfig>(c: T): T` is just an identity function that
preserves literal types. Key `AgentConfig` fields (see
`packages/agent-sdk/src/config/define-agent.ts`):

- **`id`** (required) — marketplace alias. The signed `agentId` is
  `"{agentServicePubkey}:{uuid}"`, assigned by publish — `id` is the alias, not
  the composite id.
- **`version`** (required) — semver string.
- **`entry`** OR **`engagements[]`** — *exactly one*. `entry: "./src/agent.ts"`
  is shorthand for `engagements: [{ id: "default", entry }]`. Multi-engagement
  form: each `{ id, entry, goal?, condition?, priority?, weight?, limit?,
  params?, enabled? }` (ADR-038 on-declaration selection fields, mirroring the
  `@Condition`/`@Priority`/`@Weight`/`@Limit`/`@Params` class decorators).
- **`cubbies`** — `[{ alias, migrations? }]`. `alias` must match every
  `ctx.cubby("alias")` call (build lints this).
- **`models`** — `Record<alias, modelId>`; used via `ctx.models.<alias>` (lint
  requires declared aliases).
- **`params`** — `Record<name, ParamDecl>` (`type`, `default`, `min`/`max`/`enum`;
  `"modelAlias"` must enumerate `models` keys). Resolved per Job as
  `manifest defaults ⊕ deployment ⊕ engagement` and surfaced as `ctx.params`.
- **`settings`** — declared config knobs (`string|number|boolean|url|secret`).
- **`widgets`** — `WidgetDecl[]` (see above).
- **`requiredScopes`** — vault scopes needed at connection (default `["default"]`).
- **`idleTimeout`** — duration string (`"30m"` default; `"0s"` disables). Parsed
  to ms into `manifest.idleTimeout`; on expiry the orchestrator marks the Job
  terminal and dispatches the synthetic close event.
- **`eventSchemas`** — JSON Schemas for events this agent *owns*.
- **`agentServicePubkey`** — optional; stamps full identity at build time.
- **`targeting`** — DEPRECATED (ADR-038): use on-engagement fields instead.

## Runtime & manifest model

The runtime is **CEF-agnostic**: it exposes only Web-platform globals plus one
opaque `globalThis.context`, and dispatches
`globalThis.__handlers[name](event, context)`. Two build artifacts encode this:

- **`bundle.js`** — an IIFE (esbuild output) that imports the SDK runtime shims
  (`ctx.cubby`/`.publish`/`.models`/`.close`, reading endpoints from `context`)
  and **installs `globalThis.__handlers`**: a *flat* map of handler functions.
  Event handlers are keyed `"<engagementId>::<method>"` (namespaced to stay
  collision-free across engagements); lifecycle hooks live under the fixed,
  NON-namespaced keys `onStart` / `onClose`.
- **`manifest.json`** — the **routing table** the orchestrator consults. Each
  engagement carries `handles{eventType -> handlerName}` (namespaced) and
  `hooks[]` (`"start"`/`"close"`). The orchestrator returns handler-name strings
  verbatim from `HandlerFor`, and AR looks them up directly on `__handlers`.

Gotchas:
- Only one engagement per agent may own each lifecycle hook — `cef build` errors
  on an `onStart`/`onClose` collision (hooks aren't namespaced). Event method
  names *may* be reused across engagements (they are namespaced).
- No Node built-ins. Use global `fetch` for HTTP, `ctx.cubby` for state,
  `globalThis.crypto` (WebCrypto) instead of `node:crypto`. `cef build` bans
  `fs`/`net`/`http`/`child_process` and sync DB/network packages.
- Use `console.*` for plain logging (the runtime captures it); `ctx` is only for
  CEF-specific calls.

## build → push → publish → deploy

1. **`cef build`** — lints the entry (see below), bundles with esbuild, emits
   `dist/<agentId>/{bundle.js, manifest.json}`. Identity/signature fields stay
   blank unless `agentServicePubkey` is set.
2. **`cef push`** — uploads the bundle and each widget dir to DDC as
   content-addressed objects/DAGs; stamps `{ bucketId, cid }` refs into the
   manifest. (`--as-pubkey <hex>` overrides identity.)
3. **`cef publish`** — signs the manifest and registers it with the Marketplace
   API (stub pending the API). Fills `agentId`/`agentServicePubkey`/`signature`.
4. **deploy** — a deployment record selects an engagement + pins a version; the
   orchestrator resolves the manifest via the bucket-backed CDN registry and
   dispatches Jobs against `__handlers`.

Build-time lints (mirror `@cef-ai/eslint-plugin`): `@OnEvent(...)` and
`ctx.publish(t, ...)` first args must be string literals; `ctx.cubby("alias")`
must be a literal AND a declared alias; `ctx.models.X` must reference a declared
model alias; banned imports abort the build.

Support commands: **`cef typegen`** regenerates `.cef/generated.d.ts`
(populates `KnownEventTypes` for `@OnEvent`/`ctx.publish` autocomplete) and
`cef.lock.json`. **`cef inspect <agentDir>`** runs the built bundle in `node:vm`
and reports exposed `__handlers` names vs. manifest routing targets.
