# Agent code: engagements, events, lifecycle

A CEF agent is one or more **engagements** — goal-bound code units. Each
engagement is a class **default-exported** from its own module, decorated with
`@Engagement(...)`. Event and lifecycle handlers are methods on that class. All
decorators come from `@cef-ai/agent-sdk`; types are imported `type`-only.

## The default-exported class

```ts
import {
  Engagement,
  OnEvent,
  OnStart,
  OnClose,
  type Context,
  type Event,
} from "@cef-ai/agent-sdk";

interface UserMessage { text: string }

@Engagement({
  id: "default",
  goal: "echo back any user message and persist to history",
})
export default class Echo {
  @OnStart
  async onStart(/* ctx: Context */) {
    console.info("started"); // plain logging → sandbox console; runtime captures it
  }

  @OnEvent("user_message")
  async onMessage(event: Event<UserMessage>, ctx: Context) {
    await ctx.cubby("history").exec(
      "INSERT INTO messages(id, text, ts) VALUES (?, ?, ?)",
      [Date.now(), event.payload.text, event.ts],
    );
    await ctx.publish("ack", { for: event.id });
  }

  @OnClose
  async onClose(_ctx: Context, reason: string) {
    console.info("closing", { reason });
  }
}
```

`@Engagement({ id, goal })` is required to mark the class as an engagement (`id` is
its stable identifier within the agent; `goal` is human-readable intent). Classes
using only the method decorators are transitional.

## Decorators

| Decorator | Applies to | Purpose |
|-----------|-----------|---------|
| `@Engagement({id, goal})` | class | Marks the engagement; sets id + goal. |
| `@OnEvent("type")` | method | Handler for inbound events of that type. |
| `@OnStart` / `@OnStart()` | method | Lifecycle: called once before any events. |
| `@OnClose` / `@OnClose()` | method | Lifecycle: called on shutdown. |

`@OnStart`/`@OnClose` support both bare and factory (`()`) forms — both land the
same metadata. `@OnEvent` requires the call form with a string literal.

**Handler invocation signatures** (the runtime calls with a **fresh instance and
fresh `ctx` per call** — never store per-request state on `this`):

- Event: `method(event, ctx)`
- `@OnStart`: `method(ctx)`
- `@OnClose`: `method(ctx, reason)` — `reason` is a string, defaulting to
  `"closed_by_agent"`. Semantic values (`OnCloseReason`): `"revoked"`,
  `"idle_timeout"`, `"closed_by_agent"`, `"failed"`.

**Gotcha — duplicate `@OnEvent` for the same type:** first handler wins, a
`console.warn` fires at runtime, and build tooling errors. One handler per event
type per engagement.

## The `Event<P>` type

```ts
interface Event<P = unknown> {
  id: string;      // event id
  type: string;    // event type string
  ts: number;      // unix-millis timestamp
  from: string;    // publisher (agent id or external source)
  payload: P;      // typed body
}
```

Type the payload with your own interface: `Event<UserMessage>`, then read
`event.payload.text`.

## The `Context` (`ctx`)

`ctx` is the **CEF-specific platform surface only**. Plain Web capabilities are NOT
mirrored — use sandbox globals directly: `console.*` for logging (runtime captures
it out-of-band) and `fetch` for HTTP. Do not expect `ctx.log`/`ctx.fetch`.

- `ctx.cubby(alias)` → `CubbyHandle` with `query(sql, params?)` and `exec(sql, params?)`
  (per-agent scoped SQLite-style storage; alias must be declared in `cef.config.ts`).
- `ctx.models` → model handles keyed by manifest-declared alias. Each is a
  `ModelHandle` with `infer(input)` (single-shot) and `stream(input)` (async
  iterable). Aliases are typed via `KnownModels` (filled by `cef typegen`):
  ```ts
  const reply = await ctx.models.llm.infer({ prompt: event.payload.text });
  await ctx.publish("assistant_message", { text: reply.text });
  ```
- `ctx.publish(type, payload)` → publish an event (awaitable).
- `ctx.settings` → frozen `ConnectionSettings` the user supplied at connect time.
- `ctx.params` → frozen effective params (manifest ⊕ deployment ⊕ engagement,
  resolved by the platform).
- `ctx.close(reason?)` → ask the runtime to close this agent.

## String-literal rules for `@OnEvent` / `ctx.publish`

Event types are plain strings and must **match the schemas declared in
`cef.config.ts`** (`eventSchemas` for inbound, publishable types). `cef typegen`
declaration-merges `KnownEventTypes`, so with typegen run you get autocompletion and
payload typing on both `@OnEvent("...")` and `ctx.publish("...", payload)`. Without
typegen, any string is accepted (untyped overload). Keep the literal in code exactly
equal to the manifest's type string.

## Single vs multi-engagement

**Single:** one file, one default-exported class (see `echo-agent`).

**Multi:** one file per engagement, each with its own default-exported class. Wire
them in `cef.config.ts` and put the selection knobs **on each engagement** —
`priority` (lower wins), `weight` (split traffic within a tier), `limit`, and a
CEL `condition`:

```ts
// src/engagements/simple.ts
@Engagement({ id: "simple", goal: "give a short conversational reply" })
export default class Simple {
  @OnEvent("user_message")
  async onMessage(event: Event<UserMessage>, ctx: Context) { /* ... */ }
}
```

```ts
// cef.config.ts
export default defineAgent({
  id: "multi-asst",
  engagements: [
    { id: "advanced", entry: "./src/engagements/advanced.ts", priority: 1, limit: { n: 10, per: "day" } },
    { id: "simple",   entry: "./src/engagements/simple.ts",   priority: 2 },
  ],
});
```

Equivalently, author the same knobs **on the class** with `@Condition` /
`@Priority` / `@Weight` / `@Limit` / `@Params` — `cef build` folds either form
into the manifest. The engagement `id` in the class decorator must match the
config entry's `id`. (A deployment can then override an engagement's
`priority`/`weight`/`limit` at deploy time.)
