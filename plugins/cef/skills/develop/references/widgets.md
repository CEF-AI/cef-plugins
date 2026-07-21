# Widgets

A **widget** is a small web UI an agent ships alongside its bundle. At runtime a
widget reads and writes the **viewing user's own vault** — as the connected
agent — through an injected `window.WidgetRuntime`. No secret is baked into the
widget; the user's key material stays in their wallet. The same widget renders
two ways: **embedded** inside a host (the Cere Vault) and **standalone** via a
direct link. Both talk to the SAME agent's cubby for the SAME user.

You author a widget as source, iterate against your dev vault with `cef dev`,
and ship it with `cef build` + `cef push`.

> `cef dev` and `cef build`'s widget baking require a recent `@cef-ai/cli`.
> Run `cef <cmd> --help` to confirm the flags available in your version.

## The dev loop

```sh
cef init my-agent          # scaffolds an agent with a real, queryable widget
cd my-agent && pnpm install

cef dev hello              # local dev server for the `hello` widget
#   - serves the widget on localhost
#   - wires the manifest + the widget runtime
#   - interactive wallet connect (a Cere wallet popup) — same auth as production standalone
#   - watches files → live-reloads the browser on save

# iterate: edit widgets/hello/*, save → the browser reloads automatically

cef build --env dev        # bakes the manifest + runtime <script> into dist/
cef push --env dev --bucket <bucketId> --as-pubkey <asPubkeyHex>
#   open the returned CDN URL to run the widget standalone
```

`cef dev` is the primary test loop — it authenticates against your dev vault
exactly the way a standalone link does in production, so what you see locally is
what a user gets. `cef build` bakes the widget's manifest and a `<script>` tag
for the runtime into the built output; `cef push` just uploads it.

`cef dev [widgetId]` targets one widget by its declared `id`. Endpoints for the
target environment are resolved from `--env` and baked into the manifest at
build time (there is no runtime environment picker).

### Iterating with a coding agent

If your coding agent can drive a browser (e.g. via Playwright or a browser MCP), the
loop is tight: start `cef dev`, open the printed URL, then edit the widget —
**the page live-reloads on save** — and screenshot / read the console to verify
the render and catch runtime errors, without manual refreshes.

The only human step is the **first** wallet connect (a person approves the Cere
wallet popup once). The embed-wallet keeps its session in its own origin, so
each live-reload **reconnects silently** — after that one approval the agent
iterates on the fully-connected widget: edit → auto-reload → screenshot the live
cubby data + read the console, no further popups. So an agent can verify the
render, layout, the not-connected CTA, console errors (a bad query / missing
`WidgetRuntime` call), *and* the connected/live-data state once a human has done
the initial connect. (If your wallet setup re-prompts on reload or the session
expires, the human re-approves — verify the behavior once.)

## Declaring a widget

Add a `widgets: [...]` array to `defineAgent(...)`. Each entry names a widget,
picks a **kind**, points at the widget's source `dir` + `entry`, and (for every
kind except `custom`) declares the **named queries** it runs against a cubby.

```ts
import { defineAgent } from "@cef-ai/agent-sdk/config";

export default defineAgent({
  id: "hiring",
  version: "1.0.0",
  entry: "./src/agent.ts",
  cubbies: [{ alias: "history", migrations: "./migrations/history" }],
  widgets: [
    {
      id: "candidates",                 // stable, unique within the agent; the dist subdir + `cef dev` target
      name: "Candidate List",           // display name
      description: "Recent candidates", // optional display metadata
      kind: "list",                     // typed widget kind (see below)
      cubbyAlias: "history",            // the cubby this widget queries
      queries: [                        // named SQL, referenced by id from the widget
        {
          id: "recent",
          label: "Recent candidates",
          sql: "SELECT id, name, stage, ts FROM candidates ORDER BY ts DESC LIMIT ?",
          timeoutMs: 5000,
        },
      ],
      config: { columns: ["name", "stage"] },  // kind-specific config, carried verbatim
      dir: "./widgets/candidates",      // path (relative to cef.config.ts) to the built widget source
      entry: "candidates.html",         // entry file within dir
    },
  ],
});
```

### Field reference

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Stable id, unique within the agent. The `cef dev` target and the dist subdir name. |
| `kind` | yes | One of the widget kinds below. Selects the typed config + rendering. |
| `dir` | yes | Path (relative to `cef.config.ts`) to the widget's source directory. |
| `entry` | yes | Entry file within `dir`, e.g. `"candidates.html"`. |
| `queries` | yes, except `kind: "custom"` | Named SQL run against the cubby: `{ id, label?, sql, timeoutMs? }[]`. |
| `cubbyAlias` | for query-backed kinds | Cubby alias the widget reads. Must be a declared cubby. |
| `name` / `description` | no | Human-friendly display metadata. |
| `config` | no | Kind-specific configuration, carried verbatim into the manifest. |
| `events` | no | Event types the widget subscribes to for live updates. |

## Widget kinds

Every widget declares a `kind`. Each kind has a typed `config` and a standard
rendering, so you describe *what* to show rather than hand-rolling the UI:

| Kind | Shows |
|---|---|
| `record` | A single row — one entity's fields (one query returning one row). |
| `list` | Rows from a query as a list/table. |
| `dashboard` | Several queries composed into panels/metrics. |
| `submit` | A form that `publish()`es an event to the agent. |
| `conversation` | A chat-style stream of events for a conversation context. |
| `composite` | A layout combining several of the above. |
| `custom` | You own the rendering. No `queries` requirement — call `WidgetRuntime` yourself. |

Reach for a built-in kind first; drop to `custom` only when the standard kinds
can't express the UI you need.

## `window.WidgetRuntime`

The runtime is injected for you — by `cef dev` locally, by `cef build`'s baked
`<script>` in production. It resolves the viewing user's identity, connects to
their vault as the agent, and exposes:

```ts
window.WidgetRuntime.query(ref, params?)   // ref = a named query id (from queries[]) OR raw SQL
                                            //   → { columns, rows, meta }; params are BOUND, never interpolated
window.WidgetRuntime.publish(type, payload, context?)   // → { eventId }
window.WidgetRuntime.identity()             // → { publicKey, status }
window.WidgetRuntime.agentStatus()          // → "connected" | "disconnected"
window.WidgetRuntime.connect()              // ensure a user session (interactive connect when standalone)
window.WidgetRuntime.connectAgent()         // the "install" step: connect this agent for this user
window.WidgetRuntime.onIdentityChange(cb)   // subscribe to identity/status changes
```

`query(ref, params?)` takes either a **named query id** from your `queries[]`
declaration or **raw SQL**, plus a `params` array bound to `?` placeholders —
never string-concatenate values into SQL. It returns a columnar
`{ columns, rows, meta }`.

### Example — a `list` widget

`candidates.html`:

```html
<!doctype html>
<meta charset="utf-8" />
<title>Candidate List</title>
<div id="root">Loading…</div>
<script src="./candidates.js"></script>
```

`candidates.js`:

```js
const root = document.getElementById("root");

async function render() {
  try {
    // "recent" is the query id declared in cef.config.ts; 20 binds to the ? placeholder.
    const { columns, rows } = await window.WidgetRuntime.query("recent", [20]);
    const name = columns.indexOf("name");
    const stage = columns.indexOf("stage");
    root.innerHTML = rows.length
      ? rows.map((r) => `<div>${r[name]} — ${r[stage]}</div>`).join("")
      : "No candidates yet.";
  } catch (err) {
    if (err.name === "AgentNotConnectedError") {
      renderConnectCta();          // the agent isn't connected for this user yet
    } else {
      root.textContent = `Error: ${err.message}`;
    }
  }
}

function renderConnectCta() {
  root.innerHTML = `<button id="connect">Connect agent</button>`;
  document.getElementById("connect").onclick = async () => {
    await window.WidgetRuntime.connectAgent();   // installs the agent for this user
    render();
  };
}

render();
```

A `record` widget is the same shape with a query that returns a single row; a
`submit` widget collects form input and calls
`window.WidgetRuntime.publish("candidate_added", { name, stage })`.

### Connecting the agent (`connectAgent`)

`query()` and `publish()` require the agent to be **connected** for the viewing
user. When it isn't, `query()` throws `AgentNotConnectedError` — catch it and
render a CTA wired to `connectAgent()`. `connectAgent()` is the one-time
"install" flow: it connects the agent to the user's vault for the widget's
scope. After it resolves, retry the query. Use `agentStatus()` to check state up
front and `onIdentityChange(cb)` to re-render when the user connects,
disconnects, or the agent's status changes.

## Embedded vs standalone

The same widget runs in two modes; you write it once and the runtime handles the
difference.

- **Embedded** — the widget runs inside a host (the Cere Vault). The host
  supplies identity and signing over a bridge; the user is already
  authenticated, so `connect()` resolves immediately.
- **Standalone** — the widget is opened via a direct link. The runtime
  authenticates the user itself with an interactive Cere wallet connect (a
  popup). `cef dev` runs in this mode, so local iteration matches production
  standalone.

In both modes the widget queries the **same agent's cubby for the same user** —
the only difference is where the identity comes from. Don't assume a host is
present: rely on `WidgetRuntime` (which works in both) rather than reaching for
host-specific globals.

## The manifest

`cef build` produces a **widget manifest** that the runtime reads at boot. It
carries everything the widget needs to resolve identity, connect the agent, and
run its queries — you don't assemble it by hand:

- `widgetId`, `name`, `kind`, `config` — from your `WidgetDecl`.
- `agentId`, `scope`, `cubbyAlias` — which agent/cubby the widget binds to.
- `queries[]` — `{ id, label, sql, timeoutMs }` from your declaration.
- `wallet` — `{ appId, env }` used to build the interactive connect in standalone.
- `endpoints` — the resolved `{ vault, gar, marketplace, s3GatewayAuthInfo }`
  for the built `--env`.

The runtime **reads** a manifest; it never builds one. The CLI owns manifest
construction (from `cef.config.ts` + `--env` + the agent id), which is why the
same widget source works locally under `cef dev` and in production after
`cef build`.

### What must be in the manifest vs what can live in the widget JS

**Connection + identity must be in the manifest** — the runtime needs them
*before* your JS runs (it builds the vault client + authenticates first):
`endpoints`, `wallet`, `agentId`. These can't move into the widget logic,
because the logic depends on them already being resolved.

**`cubbyAlias` is the widget's cubby binding.** It's optional if the widget
never reads a cubby, but `query()` takes **no per-call cubby argument** — it
always runs against `manifest.cubbyAlias` — so a widget that *does* query must
declare its one cubby here. (You can't select the cubby from the `query()` call
today.)

**`queries[]` are a choice.** Declaring them powers the zero-JS declarative kinds
(`list`/`record`/`dashboard`) and makes a widget's data access reviewable. But
`WidgetRuntime.query()` also accepts **raw SQL**, so a `kind: "custom"` widget
can keep its query logic in the JS — `query("SELECT text, ts FROM messages …")`
— with no `queries[]` at all. Either way the boundary that matters is the
**scope** (agent + cubby + user), enforced by the runtime regardless of the SQL.

## Gotchas checklist

- **`kind` is required, and non-`custom` kinds need `queries`.** A query-backed
  widget with no `queries` (or no `cubbyAlias`) has nothing to render.
- **Query refs are ids or raw SQL, and params are bound.** Pass values in the
  `params` array against `?` placeholders; never interpolate into SQL.
- **Handle `AgentNotConnectedError`.** First load for a new user throws it —
  render a `connectAgent()` CTA instead of an error.
- **Endpoints are build-time baked from `--env`.** Build for the environment you
  intend to run in; there is no runtime environment switch.
- **Don't depend on a host.** Standalone has no host bridge — use
  `WidgetRuntime`, not host-only globals, so the widget works in both modes.
- **Reference siblings with `./relative` paths.** The widget is served as one
  directory; absolute/external URLs won't resolve.
- **`id` must be unique within the agent** and is what `cef dev [widgetId]`
  targets.
