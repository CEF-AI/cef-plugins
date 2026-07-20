# Widgets

A **widget** is a self-contained static web UI (an entry HTML plus sibling JS/assets) that an agent ships alongside its bundle. You declare it in `cef.config.ts`, build it into a plain directory, and `cef push` uploads the whole directory to DDC as one content-addressed **directory DAG** — next to `bundle.js`. There is no baked gateway URL: consumers resolve the widget at `<gateway>/<bucketId>/<cid>/<entry>`.

## Declaring a widget

Add a `widgets: [...]` array to `defineAgent(...)`. Each entry is a `WidgetDecl`. Minimal example:

```ts
import { defineAgent } from "@cef-ai/agent-sdk/config";

export default defineAgent({
  id: "widgetized",
  version: "1.0.0",
  entry: "./src/agent.ts",
  widgets: [
    {
      id: "dashboard",                 // stable, unique within the agent; used as the dist subdir
      name: "Hiring Dashboard",        // optional display name
      description: "Shows candidate pipeline",
      cubbyAlias: "history",           // optional: cubby the widget reads/queries against
      kind: "panel",                   // optional free-form host hint
      config: { theme: "dark" },       // optional; carried verbatim into the manifest
      queries: [                       // optional named SQL queries against the cubby
        { id: "recent", label: "Recent", sql: "SELECT 1", timeoutMs: 5000 },
      ],
      events: ["candidate_added"],     // optional: event types the widget subscribes to
      dir: "./widgets/dashboard",      // path (relative to cef.config.ts) to the BUILT widget dir
      entry: "dashboard.html",         // entry file WITHIN dir
    },
  ],
});
```

### Field reference (`WidgetDecl`)

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Stable id, unique within the agent. Used as the dist subdir name. |
| `dir` | yes | Path relative to `cef.config.ts` of the **already-built**, self-contained widget directory. |
| `entry` | yes | Entry file within `dir`, e.g. `"dashboard.html"`. Resolved at `<gateway>/<bucketId>/<cid>/<entry>`. |
| `name` / `description` | no | Human-friendly display metadata. |
| `cubbyAlias` | no | Cubby alias the widget reads/queries against. |
| `kind` | no | Free-form widget-kind hint for the host. |
| `config` | no | Arbitrary widget-specific config, carried verbatim into the manifest. |
| `queries` | no | Named SQL queries: `{ id, label?, sql, timeoutMs? }[]`. |
| `events` | no | Event types the widget subscribes to. |

All fields except `id`/`dir`/`entry` are metadata copied verbatim into the manifest.

## Building a self-contained widget dir

The directory must be **fully self-contained**: the entry HTML references its siblings via `./relative` paths only (no external CDNs, no absolute URLs), because everything is served under one directory CID. A "hello-world" widget needs no framework and no runtime — just HTML + JS.

Real fixture (`packages/cli/test/fixtures/widget-agent/widgets/dashboard/`):

`dashboard.html`:
```html
<!doctype html>
<meta charset="utf-8" />
<title>Hiring Dashboard</title>
<div id="root"></div>
<script src="./app.js"></script>
```

`app.js`:
```js
document.getElementById("root").textContent = "hello from widget";
```

That's a complete, valid widget. Keep the starter widget plain like this — do not pull in the widget runtime until you actually need Vault access.

> The CLI does NOT bundle or transform your widget. `dir` must already be built by the time you run `cef push`. If your widget needs a build step (bundler, TS), run it as the agent's own `build:widgets` script before `cef build`.

## How `cef build` and `cef push` handle it

- **`cef build`** copies whatever is in each `dir` into `dist/<agentId>/widgets/<id>/` (fresh each build — stale files are removed first). It also emits one `ManifestWidget` per declared widget with a **placeholder** `dag: { bucketId: "", cid: "" }`. If `dir` is missing, or the `entry` file isn't found inside it, build prints a WARNING but still emits the manifest entry.
- **`cef push`** reads each staged `dist/<agentId>/widgets/<id>/` directory, uploads it to DDC as one content-addressed directory DAG, and stamps the real `{ bucketId, cid }` into that widget's manifest entry. A widget with no staged directory (or an empty one) is skipped with a WARNING.

So the flow is: `build:widgets` (your own) → `cef build` (stage into dist) → `cef push` (upload as DAG + stamp ref).

## Optional: `window.WidgetRuntime` (Vault-native widgets)

When a widget is loaded inside the Cere Vault host, an injected runtime (`@cef-ai/widget-runtime`, emerging) exposes `window.WidgetRuntime` for reading/writing the **viewing user's** own Vault as the connected agent — no baked secret, key material stays in the wallet iframe. Core surface:

```ts
window.WidgetRuntime.query(ref, params?)   // ref = a named query id (from queries[]) or raw SQL
                                            //   → { columns, rows, meta }; params are BOUND, never interpolated
window.WidgetRuntime.publish(type, payload, context?)  // → { eventId }
window.WidgetRuntime.connectAgent()         // the "install" flow: getAgent + vault.agents.connect()
window.WidgetRuntime.identity() / .connect() / .agentStatus() / .onIdentityChange(cb)
```

`query()` throws `AgentNotConnectedError` when the agent isn't connected for this user — catch it to render a "Connect / Install" CTA wired to `connectAgent()`.

Gotchas:
- The runtime is injected by the upload/host step, not by `cef push`. Opened directly in a browser (no Vault host) it assigns `window.WidgetRuntime` synchronously but `query()`/`publish()` reject after an ~8s handshake deadline and render "Open this widget inside the Cere Vault."
- Keep starter/hello-world widgets plain and self-contained. Only reach for the runtime once the widget genuinely needs Vault data.

## Gotchas checklist

- `id`, `dir`, and `entry` are all required; `id` must be unique within the agent (build errors on duplicates).
- `dir` must be **built already** — the CLI copies, it does not compile or bundle.
- Reference siblings with `./relative` paths only; absolute/external URLs won't resolve under the directory CID.
- A missing `dir` or `entry` produces a WARNING (not an error) at build; `cef push` then skips that widget — so a "successful" build can still ship no widget. Check push output for `widget <id>: dir DAG <cid> (<n> files)`.
- No gateway URL is baked in; consumers build `<gateway>/<bucketId>/<cid>/<entry>` from the manifest ref.
