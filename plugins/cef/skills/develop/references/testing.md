# Testing with @cef-ai/testing

`@cef-ai/testing` is a deterministic, in-process simulator for CEF agents.
Use `testAgent` for single-agent unit tests and `testPlatform` for
multi-agent / app-driven tests. Tests run under Vitest.

```sh
pnpm add -D @cef-ai/testing
```

```ts
import { testAgent, testPlatform, createModelMock } from "@cef-ai/testing";
```

## `testAgent(AgentClass, opts)`

Constructs an in-memory harness around a single agent class. It wires
`ctx.cubby` to per-alias SQLite, `ctx.models[alias]` to stubs, and
`ctx.publish` to a collector you can assert on, and installs a URL-pattern
mock as the global `fetch` for the duration of each invocation.

`opts` (all optional): `cubbies`, `models`, `settings`, `params`,
`clock: { now }`. A cubby spec is either `"alias"` (no schema) or
`{ alias, migrations }` where `migrations` is a directory of SQL files.

### Minimal passing test

```ts
import { describe, it, expect, afterEach } from "vitest";
import path from "node:path";
import { fileURLToPath } from "node:url";
import { testAgent, type TestHarness } from "@cef-ai/testing";
import Echo from "../src/agent.js";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const HISTORY_MIGRATIONS = path.resolve(__dirname, "../migrations/history");

describe("echo agent", () => {
  let h: TestHarness | undefined;
  afterEach(() => h?.dispose()); // release SQLite handles; idempotent

  it("acks user_message and persists to history", async () => {
    h = testAgent(Echo, {
      cubbies: [{ alias: "history", migrations: HISTORY_MIGRATIONS }],
    });

    // dispatch returns ONLY the publishes from this call
    const events = await h.dispatch({
      type: "user_message",
      payload: { text: "hello" },
    });
    expect(events.map((e) => e.type)).toContain("ack");

    // assert on cubby rows
    const rows = await h
      .cubby("history")
      .query<{ text: string }>("SELECT text FROM messages");
    expect(rows.map((r) => r.text)).toEqual(["hello"]);
  });
});
```

### Asserting a published event's payload

`dispatch()` returns the events published during that call. To assert a
specific event carried the right body, `find` it and read `.payload` (type it
with a cast, or declare the event in `eventSchemas` + run `cef typegen` for real
types):

```ts
const events = await h.dispatch({ type: "feedback", payload: { rating: 5 } });

const ack = events.find((e) => e.type === "feedback_ack");
expect(ack).toBeDefined();
expect((ack!.payload as { rating: number }).rating).toBe(5);
```

### Lifecycle (`@OnStart` / `@OnClose`)

`start()` runs the agent's `@OnStart` hook; `close(reason)` runs `@OnClose`.
Observe state via `h.lifecycle`.

```ts
await h.start();
expect(h.lifecycle.state).toBe("started");
await h.close("idle_timeout");
expect(h.lifecycle.state).toBe("closed");
expect(h.lifecycle.terminalReason).toBe("idle_timeout");
```

Note the two distinct "close" concerns: `h.close(reason)` drives the
lifecycle hook, while `h.dispose()` frees test resources (SQLite handles).

### Other `TestHarness` members

- `published` — cumulative log of every publish since construction (vs.
  `dispatch()`'s return, which is scoped to that one call).
- `runInCubby(alias, (handle, rawDb) => …)` — run with both the
  `CubbyHandle` and the raw `better-sqlite3` DB for synchronous asserts.
- `models` — the model-handle map attached to `ctx.models`.
- `fetchMock()` — the URL-pattern mock installed as the global `fetch`.
- `advanceTime(ms)` / `now()` — manipulate the injected clock.
- `snapshot()` / `restore(snap)` — capture/restore cubbies + clock + log.
- `replay(jsonlPath)` — feed a JSONL fixture of events through the harness.

## Model mocks

Pass either a plain `ModelHandle` (`{ infer, stream }`) or a
`createModelMock()` handle via `opts.models`. The mock matches inputs by
`JSON.stringify` deep-equality; register pairs with
`.expect(input).respond(output)`:

```ts
const llm = createModelMock();
llm.expect({ prompt: "hi" }).respond({ text: "hello back" });

h = testAgent(Assistant, { models: { llm } });
// inside the agent, ctx.models.llm.infer({ prompt: "hi" }) resolves to
// { text: "hello back" }; llm.calls records every invocation.
```

`stream()` yields each element when the response is an array, otherwise
yields the single value once. An `infer`/`stream` with no matching
expectation throws — so a wrong or missing prompt fails loud.

## `testPlatform(opts)` — multi-agent

Lifts the harness into a multi-agent world: each agent runs in its own
simulator, a shared vault routes events between them, and the test drives
interactions through the vault (optionally as a specific user/wallet).
Use it when an agent talks to peers or when an app client publishes into
the vault.

```ts
import { testPlatform, testWallet, createModelMock, type TestPlatform } from "@cef-ai/testing";
import Assistant from "../agent/src/assistant.js";
import { sendAndRead } from "../app/send-and-read.js";

let p: TestPlatform | undefined;
afterEach(() => p?.dispose()); // tears down every per-agent simulator

const llm = createModelMock();
llm.expect({ prompt: "hi" }).respond({ text: "hello back" });

p = testPlatform({
  users: { alice: { wallet: testWallet("alice") } },
  agents: {
    assistant: {
      source: Assistant,
      cubbies: [{ alias: "history", migrations: HISTORY_MIGRATIONS }],
    },
  },
  models: { llm },
});

const alice = p.user("alice");
await alice.vault.agents.connect({ agentId: "assistant" });

const reply = await sendAndRead(alice.vault, "default", "default", "hi");
expect(reply).toBe("hello back");

// assert on a specific agent's cubby
const rows = await p.runInCubby("assistant", "history", (h) =>
  h.query<{ role: string; text: string }>(
    "SELECT role, text FROM messages ORDER BY id",
  ),
);
expect(rows).toEqual([
  { role: "user", text: "hi" },
  { role: "assistant", text: "hello back" },
]);
```

- `p.vault` — simulated vault: `agents.connect`, event streams, and
  scoped per-conversation channels.
- `p.user(id)` — the per-user accessor (`.vault` scoped to that wallet).
- `p.runInCubby(agentId, alias, fn)` — assert on one agent's cubby.
- `mockAgent(spec)` — a peer descriptor to stand in for an agent you don't
  want to pull into the test bundle. (`fromMarketplace(url)` throws in v1.)

## Gotchas

- Agents call the **global `fetch`** and log with **`console.*`** — there is
  no `ctx.fetch` or `ctx.log`. The harness swaps the global `fetch` only
  during an invocation and restores it after (nesting-safe), so the stub
  never leaks between dispatches.
- `cubby(alias)` throws if the alias wasn't declared in `opts.cubbies`;
  duplicate aliases throw at construction.
- The agent class must carry `@OnEvent`/`@OnStart`/`@OnClose` decorators —
  a bare class with no metadata throws at construction.
- `settings` and `params` are frozen when installed.
- Always `dispose()` in `afterEach` to release SQLite handles.
