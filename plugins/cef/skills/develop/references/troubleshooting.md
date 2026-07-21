# Troubleshooting & debugging a CEF agent

Agents run server-side per **Job**. You can't attach a debugger, so **logging
and the local simulator are your two windows**. Design for that from the start.

## Where output goes

`console.log` / `console.info` / `console.error` from your agent go to the
**sandbox console → the platform/ROC logs**. That is the only runtime signal
from a deployed agent. There is no `ctx.log` — use `console.*`.

## Logging that is actually debuggable

- **Fold the error text into the message string.** The sandbox captures the
  **first** console arg as the log `message` (via `.String()`) and
  **JSON-serializes the rest** into a separate `args` field. The gotcha:
  `JSON.stringify(new Error(...))` is `"{}"` — an `Error`'s `message`/`stack`
  are non-enumerable — so passing the error as a *second* arg records an empty
  `{}` and hides the cause:

  ```ts
  console.error("failed to persist message", err);   // ✗ args: [{}] — err lost
  ```

  Put the detail in the first arg, where `.String()` preserves it
  (`String(anError)` → `"Error: <message>"`):

  ```ts
  const detail = err instanceof Error ? err.message : String(err);
  console.error(`[history] failed to persist message: ${detail}`);   // ✓
  ```

  (A *string* or plain object as a later arg serializes fine — it's specifically
  `Error` objects that collapse to `{}`.)

- **Prefix logs** with the subsystem (`[history]`, `[reply]`) so they're
  greppable in a busy log.
- **Log enough context** to reproduce (which operation, which keys/ids) — but
  never secrets or user PII.

## Handlers must not throw

An unhandled error in an `@OnEvent` / `@OnStart` handler can **wedge the Job**
(its queued tasks get stuck). Treat every handler as adversarial-input-facing:

- **Guard the payload** — it can be missing, empty, or the wrong shape. Validate
  before use; reply with a clear message instead of throwing.
- **Wrap side effects** (cubby writes, `publish`, model calls) in `try/catch`,
  log the real error, and degrade gracefully.

## Debug locally FIRST — the simulator

Before push → deploy → read-logs, reproduce with **`@cef-ai/testing`**
(`testAgent`). It runs your handler in-process and surfaces the **real error +
stack trace immediately** — the fastest loop by far.

```ts
import { testAgent } from "@cef-ai/testing";
// drive the exact event that failed; assert the cubby / reply; read the throw
```

Run `pnpm test` (or `npx vitest run`) and add a case for the failing event.

## Common failure modes

| Symptom (in logs) | Cause | Fix |
|---|---|---|
| `UNIQUE constraint failed: <table>.<col>` | Inserting a non-unique value into a PK/unique column (e.g. `Date.now()` as `id` collides same-ms) | Let SQLite assign `INTEGER PRIMARY KEY` (omit the id), or use a genuinely unique key |
| `NOT NULL constraint failed: <table>.<col>` | A bound value was `undefined`/`null` | Guard/default the value before the write |
| Widget: `AgentNotConnectedError` | The agent isn't connected for that user | Render a Connect CTA wired to `WidgetRuntime.connectAgent()`, then retry |
| Job stuck / no reply | An unguarded `throw` in a handler | Guard payloads; `try/catch` side effects; always `publish` a reply path |
| Types don't match the model / event | `cef.config.ts` changed but types are stale | Run `cef typegen` |
| `ctx.cubby(...)`: no orchestrator endpoint | Running the handler outside a Job context | Exercise it via `testAgent`, not by calling the class directly |

## Inspect state

- **Read the cubby** with a `query` (in a widget or a `testAgent` assertion) to
  see what's actually stored.
- **`cef inspect dist/<agentId>`** verifies the built bundle's handlers match the
  manifest routing (catches "event handled but nothing happened").
