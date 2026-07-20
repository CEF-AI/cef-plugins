# State: cubbies & migrations

A **cubby** is a per-agent, per-alias SQLite store. You declare it in
`cef.config.ts`, define its schema with SQL migrations, and read/write it from
handlers via `ctx.cubby(alias)`. Each declared alias is an isolated database
scoped to the agent instance.

## 1. Declare the cubby in `cef.config.ts`

Every cubby is an entry in the `cubbies[]` array: an `alias` plus an optional
`migrations` directory (path relative to `cef.config.ts`).

```ts
// cef.config.ts
import { defineAgent } from "@cef-ai/agent-sdk/config";

export default defineAgent({
  id: "echo",
  version: "0.1.0",
  entry: "./src/agent.ts",
  cubbies: [{ alias: "history", migrations: "./migrations/history" }],
  lifecycle: "per-stream",
});
```

`CubbyDecl` shape (from `packages/agent-sdk/src/config/define-agent.ts`):

```ts
interface CubbyDecl {
  alias: string;
  /** Directory of *.sql migrations, relative to cef.config.ts. */
  migrations?: string;
}
```

## 2. Write SQL migrations

Put one or more `*.sql` files in the migrations directory. They are read,
**sorted lexicographically**, and applied in order — so use a numeric prefix
(`001-init.sql`, `002-add-index.sql`) to pin ordering. A migration file may
contain multiple statements (applied via SQLite `db.exec`).

```sql
-- migrations/history/001-init.sql
CREATE TABLE messages (
  id   INTEGER PRIMARY KEY,
  text TEXT NOT NULL,
  ts   INTEGER NOT NULL
);
```

Migrations are additive and forward-only: add a **new** numbered file for each
schema change rather than editing an already-applied one.

## 3. Read / write from handlers

`ctx.cubby(alias)` returns a `CubbyHandle`
(`packages/agent-sdk/src/types.ts`):

```ts
interface CubbyHandle {
  query<T = unknown>(sql: string, params?: unknown[]): Promise<T[]>;
  exec(
    sql: string,
    params?: unknown[],
  ): Promise<{ changes: number; lastInsertRowid: number | bigint }>;
}
```

- **`.exec(sql, params)`** — a single DML/DDL statement with bound params;
  returns `{ changes, lastInsertRowid }`.
- **`.query<T>(sql, params)`** — returns an array of row objects. The runtime
  receives a columnar response (`{ columns, rows }`) and zips it into
  `T[]` for you.

Both are `async` — always `await`. Use `?` placeholders and pass values in the
`params` array; never string-concatenate SQL.

```ts
// src/agent.ts
import { Engagement, OnEvent, type Context, type Event } from "@cef-ai/agent-sdk";

@Engagement({ id: "default", goal: "echo + persist to history" })
export default class Echo {
  @OnEvent("user_message")
  async onMessage(event: Event<{ text: string }>, ctx: Context) {
    await ctx
      .cubby("history")
      .exec("INSERT INTO messages(id, text, ts) VALUES (?, ?, ?)", [
        Date.now(),
        event.payload.text,
        event.ts,
      ]);
    await ctx.publish("ack", { for: event.id });
  }
}
```

Reading back:

```ts
const rows = await ctx
  .cubby("history")
  .query<{ text: string }>("SELECT text FROM messages ORDER BY ts");
```

## 4. The alias must be a declared string literal

The `@cef-ai/cubby-declared-alias` ESLint rule
(`packages/eslint-plugin/src/rules/cubby-declared-alias.ts`) enforces two
things on `ctx.cubby(...)`:

1. The first argument **must be a string literal** — not a variable or
   expression. `ctx.cubby(someVar)` is a lint error (`notLiteral`).
2. When configured with the declared aliases, the literal **must name a cubby
   declared in `cef.config.ts` `cubbies[]`** — otherwise `undeclared`.

```ts
ctx.cubby("history")   // ok — declared literal
ctx.cubby(alias)       // error: notLiteral
ctx.cubby("scratch")   // error: undeclared (not in cubbies[])
```

Keeping the alias a literal lets the toolchain statically verify that every
cubby you touch has a declared schema.

## 5. Testing cubbies

The test harness gives each alias an isolated in-memory SQLite database and
applies the same migration files, so tests exercise the real schema. Declare
the same alias + migrations dir you use in `cef.config.ts`:

```ts
import { testAgent } from "@cef-ai/testing";
import Echo from "../src/agent.js";

const HISTORY_MIGRATIONS = path.resolve(__dirname, "../migrations/history");

const h = testAgent(Echo, {
  cubbies: [{ alias: "history", migrations: HISTORY_MIGRATIONS }],
});
await h.dispatch({ type: "user_message", payload: { text: "hello" } });

const rows = await h
  .cubby("history")
  .query<{ text: string }>("SELECT text FROM messages");
// rows -> [{ text: "hello" }]
```

## Gotchas

- **`exec` is single-statement.** In handlers, one statement per `.exec()`
  call (params bind via `prepare().run()`). Multi-statement scripts are only
  for migration files.
- **Missing migrations dir throws.** In tests a non-existent `migrations` path
  errors at cubby construction — point it at the real directory.
- **Lexicographic ordering.** `10-*.sql` sorts before `2-*.sql`; zero-pad
  numeric prefixes.
- **Alias is scoped, not shared.** Two aliases are two separate databases;
  there is no cross-alias query.
