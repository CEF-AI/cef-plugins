# Deploying: the `deployments/` folder, `cef deploy`, and ROC setup

`cef build` → `cef push` gets your agent's manifest and bundle into the DDC
registry, but the CEF platform **does not select or serve it until a deployment
targets it**. `cef deploy` is the step that makes an agent live and testable
(including in a sandbox). Marketplace **publish is optional and comes last** — it
only lists the agent's card for discovery and can be skipped entirely for debug,
test, or sandbox use.

Two setup actions still happen in the ROC console (they can't be done from the
CLI): creating the Agent Service and minting a DDC registry access token. This
file covers those, the `deployments/` folder, the `cef deploy` command, and what
the skill should do when a prerequisite is missing.

## The `deployments/` folder is the source of truth

An agent project declares its deployment records as files: one record per
`deployments/<name>.jsonc`, where the filename stem is the record `name`. This
mirrors the `migrations/<cubby>/` and `widgets/<id>/` conventions. `cef deploy`
reads the whole folder, assembles the deployment SET, and applies it to the
target environment.

### Record fields (in-file)

| Field | Type | Required | Meaning |
|---|---|---|---|
| `priority` | int | yes | **Lower wins.** Selects which record applies when several match. |
| `targeting` | string | yes | A CEL expression over `vault.id` / `vault.scope` / `event.type`. Empty string `""` = always eligible. |
| `version` | string | yes | A semver (e.g. `1.4.0`) or the literal `"latest"`. |
| `weight` | int ≥ 1 | no (default 1) | Splits traffic **within a priority tier** — this is how you run an A/B. |
| `enabled` | bool | no (default true) | Set false to keep a record on disk but inactive. |
| `params` | object | no | Parameters passed to the agent for this record. |
| `engagements` | object | no | Per-engagement overrides. |
| `expiresAt` | RFC3339 | no | Drops the record after this time — handy for sandbox cleanup. |

### SET rules (the folder must satisfy)

- At least one record.
- **Exactly one** record with empty `targeting` — the mandatory
  default/fallthrough, so selection is never empty.
- Unique lowercase-slug `name`s.
- `weight >= 1` on every record.
- A non-empty `version` on every record.

### Audiences & A/B

`targeting` decides *who* a record applies to. Among the records that match a
given vault/event, the **lowest `priority` tier wins**; if that tier holds more
than one record, traffic splits between them by `weight` (an A/B split). Give the
default (empty-`targeting`) record a HIGH priority number — e.g. `99` — so it is
the fallthrough and more specific, lower-priority records win when present.

> `version: "latest"` **fails closed** if `latest` was never pushed. Prefer a
> pinned semver for anything that must run.

### Example files a scaffolded project ships

```jsonc
// deployments/default.jsonc  — the mandatory fallthrough
{
  "priority": 99,
  "targeting": "",
  "version": "latest",
  "weight": 1
}
```

```jsonc
// deployments/sandbox-canary.jsonc  — a pinned build for sandbox vaults
{
  "priority": 10,
  "targeting": "vault.scope == 'sandbox'",
  "version": "1.4.0",
  "weight": 1
}
```

Here a sandbox-scoped vault matches both records; priority 10 beats priority 99,
so it gets the pinned `1.4.0`. Everyone else falls through to the default.

## `cef deploy` — the command

```bash
npx cef deploy [--version <v>] [--endpoint <url>] \
               [--as-pubkey <hex>] [--author <who>] [--note <msg>] [--dry-run]
```

- Reads `deployments/` and applies the assembled SET. **Errors if the folder is
  absent or empty** — there is nothing to deploy.
- `--endpoint` (or `$CEF_ENDPOINT`) selects WHERE to apply, defaulting to the dev
  environment. It is **orthogonal** to the deployment files: the same
  `deployments/` folder applies to any environment. Do **not** put environment
  names inside deployment files.
- `--version <v>` overrides the `version` in the applied records without
  editing files — the everyday "pushed a new build, roll it out" step.
- The whole folder is applied as one set (declarative desired state): a record
  deleted locally disappears remotely on the next deploy.
- `--author` / `--note` annotate the applied set for audit.
- `--dry-run` prints what would be applied without applying it. Use it to confirm
  the assembled set before a real deploy.

The ROC console **Deploy** button applies the same deployment set; either path
works.

## ROC setup step 1 — Agent Service + AS pubkey

`cef push --as-pubkey <hex>`, `cef deploy --as-pubkey <hex>`, and
`cef publish --as-pubkey <hex>` all require the on-platform **Agent Service
pubkey**. This is the *routing* identity the platform looks up to deliver events;
it is provisioned by the platform and shown in the ROC console. It is distinct
from the AS Ed25519 keypair that signs the publish envelope (`cef keypair
generate`).

In ROC: open (or create) your **Agent Service**; its details dialog shows the AS
pubkey (hex). Copy it — you pass the same value to every `--as-pubkey` flag.

> Gotcha: an agent can `cef publish` fine yet be **unroutable** if its AS pubkey
> does not match a provisioned Agent Service. The `connect` check enforces that
> the `agentId` begins with the provisioned AS pubkey — so always use the real
> provisioned pubkey, not just any local keypair, for anything that must run.

## ROC setup step 2 — DDC registry access token

`cef push` writes the manifest + bundle into your Agent Service's **DDC registry
bucket** (the bucket *is* the registry). To authorize the write without handing
out the bucket's raw secret phrase, mint a token in ROC:

1. Open the **ROC console** → your **Agent Service** → **Access** tab.
2. It shows the AS's registry `bucketId` and a "generate access token" control
   with a selectable validity window.
3. Generate and **copy immediately — the token is shown once.** Note the
   `bucketId` too — you pass it to `cef push --bucket`.

The token is a base58, full-access (GET/PUT/DELETE) DDC token scoped to that
bucket, signed by the bucket-owning wallet. It is a **bearer** token any holder
can use until it expires; it is stateless and **cannot be individually
revoked** — it only expires (default ~30 days). Prefer a short window and
regenerate on exposure.

```bash
export CEF_DDC_ACCESS_TOKEN="<base58 token from ROC>"
cef push --bucket <bucketId> --as-pubkey <asPubkeyHex>
```

`cef push` requires **exactly one** of `--access-token` /
`$CEF_DDC_ACCESS_TOKEN` or `--secret-phrase` / `$CEF_DDC_SECRET_PHRASE`. Neither
→ hard error; both → hard error. The token path is the common team case (you
don't hold the phrase).

## Detect the missing prerequisite → hand back the exact next step

Diagnose *which* gate is unmet and return the precise command or ROC step. In
flow order:

| Symptom / state | What's missing | Hand back |
|---|---|---|
| No AS keypair (`~/.cef/*.key`, `~/.config/cef/credentials`, `$CEF_AS_*`) | Publisher identity | `cef keypair generate --out ~/.cef/my-agent.key` (once; share via a secrets manager) |
| No AS pubkey to pass `--as-pubkey` | ROC setup step 1 | "Open ROC → your Agent Service → copy the AS pubkey (hex)" |
| `push` errors "no DDC authorization" | ROC setup step 2 | "Open ROC → Agent Service → Access → generate token; `export CEF_DDC_ACCESS_TOKEN=…`" (note the `bucketId` for `--bucket`) |
| `push` errors "both … provided" | Ambiguous auth | Unset one of `$CEF_DDC_SECRET_PHRASE` / `$CEF_DDC_ACCESS_TOKEN` (or drop one flag) |
| `push` errors "bundle not found" / "no built agents" / "no version" | Build not run | `cef build` first |
| `deploy` errors "no deployments" / empty folder | No records to apply | Create `deployments/default.jsonc` (priority 99, targeting `""`), then re-run |
| Pushed but a user's `connect` never gets served | Not deployed | `npx cef deploy --endpoint <url> --as-pubkey <hex>` (or ROC → the agent → **Deploy**) |
| `connect` rejected (agentId prefix mismatch) | Wrong AS pubkey | Re-push/deploy with the **provisioned** AS pubkey so `agentId` starts with it |
| Card absent from the marketplace, but the agent runs | Not published | `npx cef publish …` — optional, discovery only |

Rule of thumb: the CLI's own error strings name the missing gate ("run
`cef build` first", "no DDC authorization"). Surface that string verbatim, then
translate it into the one next command or ROC click — don't guess further ahead
than the current gate.

## The full flow (for reference)

```bash
# one-time: publisher identity (share via a secrets manager, not per-dev)
cef keypair generate --out ~/.cef/my-agent.key
# one-time per token window: copy from ROC → Agent Service → Access
export CEF_DDC_ACCESS_TOKEN="<base58 token from ROC>"

npx cef init                                            # scaffold, incl. deployments/
npx cef build
npx cef push    --bucket <bucketId>            --as-pubkey <asPubkeyHex>
npx cef deploy  --endpoint <url>               --as-pubkey <asPubkeyHex>   # makes it live
# optional, last — only if you want a marketplace listing:
npx cef publish --keystore ~/.cef/my-agent.key --as-pubkey <asPubkeyHex>
```

The `--endpoint` (or `$CEF_ENDPOINT`) defaults to the dev environment; point it
at another environment to deploy there — the same `deployments/` folder applies
either way. Select the DDC network for `push` with `--preset
MAINNET|TESTNET|DEVNET` (default `MAINNET`) or `--endpoint`.
