# Runtime constraints & lints

`cef build` runs a **syntactic lint pass over the agent's entry file before
bundling**. Any violation aborts the build with a single, actionable message
(`[cef build] <file>:<line>:<col> — <msg>`). There is no error-collection
mode — the *first* violation throws. `@cef-ai/eslint-plugin` (`recommended`
config) mirrors every rule at edit time, so your editor agrees with CI.

These are hard rules of the agent sandbox, not style preferences. Fix them at
the source; there is no override flag.

> Scope note: v1 lints the **entry file only**. Helper modules imported via
> `./` paths are not scanned syntactically, but banned imports are still
> caught at bundle time when esbuild resolves them. Keep the rules in mind
> everywhere, not just the entry.

## 1. Banned imports

Agents run in a sandbox with **no Node built-ins and no sync DB/network
libraries**. Importing any of these — via `import`, `require(...)`,
`import(...)`, or `export ... from` — fails the build.

Banned Node built-ins (matched with or without the `node:` prefix, including
subpaths like `node:fs/promises`):

```
fs, fs/promises, path, net, tls, dgram, dns, http, https, http2,
child_process, worker_threads, cluster, os, process, readline, repl,
vm, stream, zlib, buffer, crypto
```

Banned npm packages (sync DB / raw network clients):

```
pg, pg-native, mysql2, mysql, mongodb, redis, ioredis, sqlite3,
better-sqlite3, ws, node-fetch, axios
```

### What to use instead

| Instead of… | Use… |
| --- | --- |
| `http` / `https` / `node-fetch` / `axios` | the global `fetch` (`await fetch(url)`) |
| `fs`, `pg`, `redis`, `mongodb`, … (state/storage) | `ctx.cubby("alias")` (declared persistent state) |
| `node:crypto` | `globalThis.crypto` — the WebCrypto API |
| `net` / `tls` / `ws` (raw sockets) | `fetch` (HTTP only; raw sockets are not available) |
| `buffer` / `stream` | Web standards: `Uint8Array`, `TextEncoder`/`TextDecoder`, `ReadableStream` |

```ts
// BAD — build fails: banned import "node:crypto"
import { randomUUID } from "node:crypto";
const id = randomUUID();

// GOOD — WebCrypto is on the global
const id = globalThis.crypto.randomUUID();
const digest = await globalThis.crypto.subtle.digest("SHA-256", bytes);
```

```ts
// BAD — banned import "axios"
import axios from "axios";
const res = await axios.get(url);

// GOOD — global fetch
const res = await fetch(url);
const data = await res.json();
```

## 2. `@OnEvent(...)` must be a string literal

The event-type argument is read statically to build the manifest routing
table, so it cannot be a variable or expression.

```ts
// BAD — @OnEvent argument must be a string literal
const EVT = "user_message";
@OnEvent(EVT)

// GOOD
@OnEvent("user_message")
```

## 3. `ctx.publish(t, ...)` first arg must be a string literal

The literal is collected into the manifest's `publishes[]` (when not declared
in `cef.config.ts`), so it must be inline.

```ts
// BAD — ctx.publish first argument must be a string literal
ctx.publish(topic, payload);

// GOOD
ctx.publish("assistant_message", payload);
```

## 4. `ctx.cubby("alias")` — literal AND declared

The first arg must be a string literal **and** a `cubbies[].alias` declared in
`cef.config.ts`. An undeclared alias fails with the list of declared aliases.

```ts
// cef.config.ts declares: cubbies: [{ alias: "history", ... }]

// BAD — literal but not declared:
//   ctx.cubby alias "chat" is not declared in cef.config.ts cubbies[]
const c = ctx.cubby("chat");

// BAD — not a literal:
//   ctx.cubby first argument must be a string literal
const c = ctx.cubby(name);

// GOOD
const c = ctx.cubby("history");
```

Fix: add the alias to `cubbies[]` in `cef.config.ts`, or use an already-declared
one.

## 5. `ctx.models.X` must be a declared model alias

Member access on `ctx.models` must name a model alias declared under `models`
in `cef.config.ts`.

```ts
// cef.config.ts declares: models: { llm: ... }

// BAD — ctx.models.gpt is not declared in cef.config.ts models
const out = await ctx.models.gpt.complete(prompt);

// GOOD
const out = await ctx.models.llm.complete(prompt);
```

## Keeping editor and CI in sync

Enable `@cef-ai/eslint-plugin` `recommended` to catch all of the above at edit
time. The declared-alias/type rules need the sets passed as options; wire them
from `cef.config.ts`:

```js
// .eslintrc.cjs
module.exports = {
  plugins: ["@cef-ai"],
  extends: ["plugin:@cef-ai/recommended"],
  rules: {
    "@cef-ai/cubby-declared-alias": ["error", { aliases: ["history"] }],
    "@cef-ai/model-declared-alias": ["error", { aliases: ["llm"] }],
    "@cef-ai/publish-declared-type": [
      "error",
      { knownEventTypes: ["assistant_message"] },
    ],
  },
};
```

`cef typegen` refreshes `KnownEventTypes` (in `.cef/generated.d.ts`) so the
string-literal args autocomplete; run it after changing `cef.config.ts`.

> Gotcha: the lint identifies context calls by the **head identifier `ctx`**,
> by convention. Rename the context parameter (e.g. `context.cubby(...)`) and
> the alias/literal checks silently stop firing at build time — keep the
> parameter named `ctx`.
