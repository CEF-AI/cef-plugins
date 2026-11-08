# The memory bank (`ctx.memory`, `ctx.workspace`)

> **Requires `@cef-ai/agent-sdk` >= 5.13 and `@cef-ai/testing` >= 4.0, which are
> unpublished as of 2026-11-08** (CEF-AI/sdk#309). On agent-sdk 5.12 and earlier,
> `ctx.memory` has only `upsert`/`update`/`setPrivacy`/`delete`/`relation`/
> `search(match, {limit})`/`get`/`neighbours`/`countByType`, and none of the
> methods below exist. Check `node_modules/@cef-ai/agent-sdk/package.json` before
> you use them.

The memory bank is the vault's durable, privacy-classified store of
**accepted organizational knowledge**. One rule shapes every API below:

```
workflow run ──writes──▶ its own cubby table (raw, transient)
                              │ read by
                              ▼
          analyzer (counts) / harvester (statements)   ← curator connections
                              │ ctx.memory.propose → status "proposed"
                              ▼
                  bank: proposed records ── a person accepts (ROC) ──▶ accepted
                                                                         │
            workflows, Bridge agents, ROC Ask ◀── typed reads, accepted by default
```

- **A workflow never writes the bank.** It writes its own cubby table. A
  curator turns those rows into proposals, and a person accepts them.
- **Only curators write the bank, and only proposals.** A curator is an agent
  whose connection the vault owner has made a *harvester* or an *analyzer*.
- **Readers read accepted records**, and never join back to a cubby. Whatever a
  reader needs belongs in the record or its `props`.

## Which surface you use

| You are writing… | Use | Not |
|---|---|---|
| A workflow (or an agent doing work) | `ctx.cubby(alias)` for your own rows, or `ctx.workspace(alias)` to leave rows for a curator in another agent service | `ctx.memory.upsert`, or `remember`/`relate` steps |
| A **harvester** (statements a person should accept: principles, findings, decisions) | `ctx.memory.propose(...)` | `ctx.memory.upsert` (it cannot send a status) |
| An **analyzer** (counts, rates and evidence) | `ctx.memory.proposeMetric(...)`, `ctx.memory.proposeObservation(...)` | `propose` |
| A **reader** (any agent or workflow step that uses knowledge) | `ctx.memory.search` / `list` / `record`, or a `recall` step; `ctx.memory.metrics` for numbers | reading a curator's cubby |

## Workflows: write your own cubby table

```ts
// a step or handler in the workflow — raw rows, in the workflow's own cubby
await ctx.cubby("reviews").exec(
  "INSERT INTO candidate_reviews(subject_key, run_id, score, decided_at) VALUES (?, ?, ?, ?)",
  [lead.key, runId, score, new Date().toISOString()],
);
```

Design the table so a curator can read it without guessing:
- **A stable subject key per row.** Never a display name.
- **`run_id`, the workflow version, and an `at` timestamp** on every row.
- **Signals as columns with a closed vocabulary.** Never free text.
- **Outcomes in an append-only table.** Never UPDATE the decision row.

**A curator in another agent service cannot read your service's cubby.** Write
the rows to the scope's shared workspace instead:

```ts
await ctx.workspace("outreach").exec("INSERT INTO touches(...) VALUES (...)", [...]);
// addresses <vault>/<scope>/_shared/outreach; scope defaults to ctx.self.scope
const rows = await ctx.workspace("outreach", { scope: "gtm" }).query("SELECT ...");
```

`ctx.workspace` has the same `query`/`exec` as `ctx.cubby`. Any agent
authorized in that scope may use it.

### `remember` / `relate` steps in a workflow are advised against

- **On a vault running the `curators` write policy**, the bank refuses a
  workflow's write with 403.
- **On a vault running `allow`**, the write lands as an *accepted* record that
  no person reviewed, which is worse.
- **What `cef build` does about it:**
  - with agent-sdk 5.13 it **warns** on every such step (`memory-write-from-workflow`):
    "Workflows don't write the bank: write your own cubby table; a harvester
    proposes from it, and a person accepts."
  - from **agent-sdk 6.0** (planned, not released) the same step is a **build
    error** unless the node declares `curator: true`.
- **Replace the step** with a write to your cubby table, plus a curator that
  proposes from it.

## Curators: `ctx.memory.propose`

```ts
import { sourceHash, isMemoryError } from "@cef-ai/agent-sdk";

const res = await ctx.memory.propose({
  id: "principle:hiring-icp",                // C1 grammar: <type>:<key>, revisions <type>:<key>~n
  type: "principle",                          // must equal the id's type segment
  title: "Prefer candidates with shipped side projects",
  body: "Advanced 2x as often in cohort B (window ending 2026-11-01).",
  scope: "gtm",
  privacy: "internal",                        // required, never defaulted
  domain: "hiring",                           // required: an upsert without it clears it
  props: {
    source: "cubby:gtm/_shared/reviews",      // required: what you read
    source_hash: sourceHash(inputRows),        // required: fingerprint of the INPUTS
  },
  derivedFrom: ["metric:hiring-funnel:cohort-b@2026-11-01"],   // edges: this --derived_from--> each
});
// res.outcome: "written" | "revised" | "unchanged" | "decided"; res.id; res.record; res.edges
```

### What `propose` guarantees

- **It always writes `status: "proposed"`.** The input has no `status` field.
  `version` is never sent.
- **It reads the record first** (C1 §2):

  | The record is… | `propose` does | `outcome` |
  |---|---|---|
  | absent (or hidden from you) | writes it | `written` |
  | proposed, same content | nothing; draws any missing edges | `unchanged` |
  | proposed, different content | rewrites it in place | `written` |
  | rejected or superseded | nothing | `decided` |
  | accepted, **same `props.source_hash`** | nothing, **whatever the title or body say** | `decided` |
  | accepted, `source_hash` changed | writes `<id>~n+1`, with `supersedes` → the newest accepted revision | `revised` |

  **Reworded text over the same inputs never makes a revision.** That is why
  `source_hash` must fingerprint the inputs (the rows or evidence ids you read,
  and the config that shaped the claim), never the generated text.
  `sourceHash(x)` gives `sha256:<hex>` over canonical JSON with sorted keys.
  It throws a `TypeError` on values JSON would silently change: a `Map`, a
  `Set`, `NaN`/`Infinity`, a symbol, a bigint, a function or an invalid `Date`.
  Convert a `Map` or `Set` to an object or array before you hash it.
- **`opts.onDecided`** applies to an accepted record whose hash changed:
  - `"revise"`, the default, files the next revision;
  - `"skip"` writes nothing;
  - `"throw"` raises `DECIDED`.
- **It reads every write back.** The bank answers a write with the same empty
  204 whether or not it kept it: a hidden scope, privacy above your ceiling,
  or a record a person decided all look the same as success. `propose` reads
  the record and each edge back, and throws **`MemoryError` with `code:
  "NOT_LANDED"`** when the bank does not show them. Never trust a write ack.
- **It fills these props for you:** `run_id` (the job), `revision` (from the
  id), and `curator` (`<agent>@<version>`, which `cef build` bakes into the
  bundle; override it with `opts.curator`). Never set `reviewed_by` or
  `reviewed_at`: the store writes them when a person decides.
- **Edges:** `supersedes`, `derivedFrom[]`, `about[]` and `involves[]`. `in` is
  the new record and `out` is the target, so the edges read
  `new --supersedes--> old` and `claim --derived_from--> evidence`.

### Ids (C1 record grammar)

- **The shape.** `<type>:<key>` for revision 1, `<type>:<key>~<n>` for n ≥ 2.
  The key is `[a-z0-9][a-z0-9._/-]{0,119}`.
- **No lifecycle words in an id** (`proposal:`, `core:`). A curator not yet
  migrated passes `{ allowLegacyId: true }`.
- **No scope in an id.** Scope is a column.
- **Platform types are refused:** `run`, `outcome`, `person`, `domain`.
- **Helpers:** `parseRecordId`, `revisionId`, `nextRevisionId` and
  `parseMetricId`, all exported from `@cef-ai/agent-sdk`.

### Analyzers: `proposeMetric` / `proposeObservation`

```ts
await ctx.memory.proposeMetric({
  id: "metric:gtm-outreach-funnel:all@2026-11-01",   // metric:<analyzer>:<segment>@<YYYY-MM-DD|YYYY-Www>
  title: "Outreach funnel, all segments", scope: "gtm", privacy: "internal", domain: "gtm",
  props: { source: "cubby:gtm/_shared/outreach" },
  values: { subjects: 40, replied: 10, reply_rate: 0.25 },  // keys lower_snake_case, |v| ≤ 1e15
});
await ctx.memory.proposeObservation({
  id: "obs:gtm-outreach-funnel:lead-42@cfg-7",       // obs:<analyzer>:<subject>@<config_version>
  title: "...", scope: "gtm", privacy: "internal", domain: "gtm", props: { source: "..." },
});
```

- **Type and label are set for you.** The type is `metric` or `observation`,
  and the SDK adds `props.measured = true`.
- **The numbers are read back.** If the bank does not show them, the call
  throws `NOT_LANDED`.
- **A window a person decided is left alone** (outcome `decided`). Re-running
  an accepted window is safe.

### Errors

Every `ctx.memory` method throws `MemoryError`. It extends `Error` and keeps the
message prefix `[@cef-ai/agent-sdk] ctx.memory.<op> …`.

| `err.code` | Meaning | What to do |
|---|---|---|
| `INVALID` | refused before any request (bad id, missing `domain` / `source_hash`, bad privacy…) | fix the input |
| `NOT_LANDED` | acked but not kept (`err.current` is what the bank shows) | check the scope, the privacy vs the run initiator's ceiling, and whether a person decided it |
| `WRITE_REFUSED` | 403 from a policy or role rule: a non-curator write under `curators`, an analyzer id or key rule, a platform-reserved id or edge, **or `err.serverCode === "PRIVACY_ABOVE_CEILING"`**. For a policy or role refusal the server sends `SCOPE_DENIED`, so `err.serverCode` is `"SCOPE_DENIED"` while `err.code` is `WRITE_REFUSED`. Branch on `err.code` | see below |
| `SCOPE_DENIED` | 403: the scope is not one your connection writes (`err.message` lists the scopes it may write) | write to a scope the connection holds |
| `NO_INITIATOR` | 403: the run has no initiator, so it has no ceiling. Slack/webhook-triggered runs can be like this | run from an event a person published, or a schedule |
| `DECIDED` | `onDecided: "throw"` met a decided record | expected; `err.current` holds it |
| `UNSUPPORTED` | the server ignored a facet only a newer vault-api understands (`idPrefix`, or `domains` on `list`) | drop the facet until vault-api with C3 is deployed. The SDK will not filter on the client |
| `BAD_REQUEST` / `HTTP` | 400 / anything else | read `err.message` |

**`PRIVACY_ABOVE_CEILING`** applies on vault-api with contract C3. A record or
edge whose `privacy` is above **the run initiator's** ceiling is refused with 403,
and `err.serverCode` is `"PRIVACY_ABOVE_CEILING"` (`err.code` is `WRITE_REFUSED`).
Before C3 the same write landed where its writer could not see it, so `propose`
reported `NOT_LANDED`. Fix it by classifying the record at most as high as the
initiator's ceiling (`internal` is the usual choice for aggregates), or by
having a member with a higher ceiling connect or start the run.

## Readers: typed reads, accepted by default

```ts
const hits = await ctx.memory.search("pricing objection", {
  types: ["principle", "finding"], domains: ["sales"], limit: 10, include: ["props"],
});                                     // MemoryRecord[] — status "accepted" only
const page = await ctx.memory.list({ types: ["principle"], domain: "hiring" });  // newest first, page.cursor
const rec = await ctx.memory.record("principle:hiring-icp");  // one record or null; ANY status
if (rec?.status === "accepted") { /* act on it */ }
```

- **Default: accepted records only.** Pass `statuses: ["accepted", "proposed"]`
  only to *show* unreviewed items labelled as such, or to avoid re-proposing.
  Never act on a proposal as if it were knowledge.
- **`record(id)` and `neighbours(id)` return any status.** Check `status`
  before acting on a record you reached by id or by edge.
- **Filters:**
  - `types` (up to 10) and `domains` (up to 20) narrow the read, inside what
    your connection may see. They never widen it.
  - `include` adds `body` (cut to `bodyChars`), `props`, `created_at`,
    `updated_at`, or `review` (the stamp: `reviewedBy` / `reviewedAt`).
  - `idPrefix`, and `domains` on `list`, need vault-api with C3. On an older
    server the call throws `UNSUPPORTED`.
- **Read narrowly, then use what you read.** Targeted reads draw `used` edges
  from your run to the records (up to 200 per job); that is how reuse is
  measured. Cite the ids you used in your output and cubby rows.
- **Workflow `recall` step:**
  - **Parameters:** `match`, `types`, `domains`, `statuses` (default accepted),
    `include`, `idPrefix`, `limit` (default 50), `bodies`, `maxBodyChars`,
    `into`.
  - **Search or list:** with `match` it searches; with only filters it lists,
    newest first; with neither it reads nothing.
  - **Rows** carry `status`.

### Metrics are measured, never accepted

An analyzer's numbers stay `proposed` forever. Show them as **measured**, never
as accepted knowledge:

```ts
const { records } = await ctx.memory.metrics({ domain: "gtm" });  // newest period per segment
for (const m of records) {
  // m.measured === true, m.label === "measured, not reviewed"
  console.info(m.analyzer, m.segment, m.period, m.values.reply_rate);
}
const one = await ctx.memory.metric("metric:gtm-outreach-funnel:all@2026-11-01");
```

- **`metrics()` options:**
  - `domain`;
  - `analyzer` and `segment` (sent as `idPrefix`, so they need C3);
  - `keys`;
  - `latest` (default true), which keeps the newest period for each segment.
    A segment whose ids mix daily and weekly periods throws `INVALID`; read it
    with `latest: false`;
  - `limit` (default 25, max 50, because it makes one read per record);
  - `cursor`;
  - `legacy: true`, which also lists the older `metric:<scope>:<analyzer>:…`
    records.
- **Not from `search` or `list`.** Their accepted-only default never returns
  metrics.
- **Not from a `recall` step either.** `cef build` warns
  (`recall-metric-records`), because the rows come back without numbers or the
  "measured" label. Read metrics with `ctx.memory.metrics` in a code step.
- **A statement derived from metrics** ("segment X converts 2x better, so
  prefer it") is a harvester's `propose`, with `derivedFrom` pointing at the
  metric. A person accepts the statement, not the number.

## Testing (`@cef-ai/testing` >= 4.0)

- **Every `testPlatform` has its own fake bank,** `p.memory`. It runs
  vault-api's rules behind the real `ctx.memory` code.
- **Its default policy is `curators`,** so a workflow's write fails as it
  would on devnet:

```ts
const p = testPlatform({
  agents: { harvester: { source: Harvester } },
  memory: { connections: { harvester: { role: "harvester" } } },  // or { policy: "allow" }
});
p.memory.seed({ id: "metric:f:all@2026-11-01", type: "metric", scope: "gtm", privacy: "internal", status: "proposed" });
// …dispatch…
expect(p.memory.records.get("principle:hiring-icp")?.status).toBe("proposed");
p.memory.accept("principle:hiring-icp");          // as a person: accepted + review stamp
```

- **Connections:**
  - `memory.connection` sets the default; `memory.connections[agentId]` sets
    one agent's `scopes`, `ceiling`, `role` or `initiator`.
  - The default connection reaches every scope at ceiling `restricted`, with no
    role.
- **`c3: false`** models a vault-api without contract C3. Facets are ignored,
  so `UNSUPPORTED` is thrown, and an above-ceiling write lands invisibly, so
  `NOT_LANDED`.
- **A single agent:** `testAgent(Agent, { memory: {} })` gives it a bank. Without
  `memory`, every `ctx.memory` call throws.
- **Assertions:** use `p.memory.records`, `p.memory.relations`,
  `p.memory.props(id)` and `p.memory.requests` (every door call, with its body).
