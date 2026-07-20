---
name: develop
description: Author, extend, and debug a CEF agent — add engagements/event handlers, cubbies and migrations, widgets, and tests. Use when writing or changing agent code in a CEF agent project.
---

# Develop a CEF agent

A CEF agent is a TypeScript project: a declarative manifest (`cef.config.ts`)
plus a default-exported **engagement** class (`src/agent.ts`) whose methods
handle events and lifecycle. `cef build` lints + bundles it into
`dist/<agentId>/{bundle.js, manifest.json}`; the Cere Agent Runtime executes the
bundle against Web globals plus one CEF surface, `ctx`. Everything you write
lives inside that sandbox — the constraints below are hard rules, not style.

## OVERVIEW — the loop

1. **Understand the project.** Read `cef.config.ts` (identity, engagements,
   cubbies, models, widgets, params) and the entry class it points at. The
   config is the source of truth the manifest and lints are derived from.
2. **Author the agent class.** Add `@OnEvent("type")` methods and `@OnStart` /
   `@OnClose` lifecycle hooks on the `@Engagement`-decorated default export.
3. **Add state.** Declare a cubby in `cubbies[]`, write SQL migrations, read/write
   via `ctx.cubby("alias")`.
4. **Add UI (optional).** Ship a self-contained widget dir declared in `widgets[]`.
5. **Test.** `@cef-ai/testing` (`testAgent` / `testPlatform`) under Vitest, with
   the same cubby aliases + migration dirs as the config.
6. **Respect the runtime constraints on every edit** — no Node built-ins, string
   literals for `@OnEvent` / `ctx.publish` / `ctx.cubby`, declared aliases only.

After changing `cef.config.ts`, run `cef typegen` so `@OnEvent` / `ctx.publish`
literals autocomplete and type their payloads. Then `cef build` to lint + bundle.

## ROUTING — read the reference before you edit that layer

| You are… | Read | Covers |
|---|---|---|
| Orienting in an unfamiliar project | [anatomy.md](./references/anatomy.md) | file layout, `defineAgent` fields, manifest/`__handlers` model, build→push→publish→deploy |
| Writing handlers / lifecycle / multi-engagement | [engagements.md](./references/engagements.md) | `@Engagement`/`@OnEvent`/`@OnStart`/`@OnClose`, `Event<P>`, the `ctx` surface, selection decorators |
| Adding or querying persistent state | [cubbies.md](./references/cubbies.md) | `cubbies[]` decl, SQL migration ordering, `.query`/`.exec`, alias lint |
| Shipping a static UI | [widgets.md](./references/widgets.md) | `WidgetDecl`, self-contained dirs, `cef push` DAG, `window.WidgetRuntime` |
| Writing tests | [testing.md](./references/testing.md) | `testAgent`, `testPlatform`, model mocks, `dispatch`/`published`, harness members |
| Debugging a build failure or lint error | [constraints.md](./references/constraints.md) | banned imports, string-literal + declared-alias rules, ESLint wiring |

## The shape at a glance

```ts
// cef.config.ts
import { defineAgent } from "@cef-ai/agent-sdk/config";
export default defineAgent({
  id: "echo",
  version: "0.1.0",
  entry: "./src/agent.ts",                                  // OR engagements: [...]
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

## Runtime constraints (always in force — see constraints.md)

- **No Node built-ins, no sync DB/network libs.** `fetch` for HTTP,
  `ctx.cubby` for state, `globalThis.crypto` (WebCrypto) instead of `node:crypto`.
  `cef build` bans `fs`/`net`/`http`/`child_process`/`pg`/`ws`/`axios`/… .
- **Literals, statically read into the manifest.** `@OnEvent("...")`,
  `ctx.publish("...", …)`, and `ctx.cubby("...")` first args must be inline string
  literals; the cubby alias (and any `ctx.models.X`) must be **declared** in
  `cef.config.ts`.
- **Keep the context param named `ctx`** — the lints key off that identifier.
- **Fresh instance + fresh `ctx` per call** — never stash per-request state on `this`.
- **Use `console.*` for logging** (runtime captures it); `ctx` is CEF-only —
  there is no `ctx.log` / `ctx.fetch`.
- **One engagement owns each lifecycle hook**; `@OnStart`/`@OnClose` collisions
  fail the build. Event method names may repeat across engagements (namespaced).

## Dev commands

In a scaffolded project `cef` is a local dev-dependency, so run it via `npx cef …`
(bare `cef` is not on `PATH`). A `cef init` project also aliases the common ones
as npm scripts (`pnpm run build`, `pnpm test`).

```sh
npx cef typegen                 # after editing cef.config.ts: refresh .cef/generated.d.ts + cef.lock.json
npx cef build                   # lint entry + bundle → dist/<agentId>/{bundle.js, manifest.json}
npx cef inspect dist/<agentId>  # verify exposed __handlers match manifest routing
npx vitest run                  # run the @cef-ai/testing simulator tests  (or: pnpm test)
```

`cef push` (upload bundle + widget DAGs) and `cef publish` (sign + register) come
after build; see anatomy.md for the full pipeline.
