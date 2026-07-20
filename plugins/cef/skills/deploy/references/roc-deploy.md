# Deploying in ROC

`cef build` → `cef push` → `cef publish` gets your agent's manifest into the DDC
registry and its card into the marketplace — but the orchestrator **does not
select or serve it until a deployment targets it** (ADR-038). Deployment is
control-plane state, applied today as a **ROC console action** (a `cef deploy`
CLI is coming). This file covers the human-only steps in ROC that bracket the
CLI flow, plus what the skill should do when a prerequisite is missing.

## The two ROC touchpoints

The CLI flow depends on two things that only exist once you set them up in the
ROC console, and one action you take after publishing:

1. **Before `cef push`** — an Agent Service (giving you an AS pubkey) and a DDC
   registry access token.
2. **After `cef publish`** — click **Deploy** and select a version.

## Prerequisite A — Agent Service + AS pubkey

`cef push --as-pubkey <hex>` and `cef publish --as-pubkey <hex>` both require
the **on-platform Agent Service pubkey**. This is the *routing* identity the
orchestrator looks up to deliver events — provisioned by TSS-DKG on a real
cluster and shown in the ROC console. It is distinct from the AS Ed25519
keypair that signs the publish envelope (`cef keypair generate`).

In ROC: open (or create) your **Agent Service**; its details dialog shows the
AS pubkey (hex). Copy it — you pass the same value to both `--as-pubkey` flags.

> Gotcha: a card can `cef publish` fine yet be **unroutable** if
> `agentServicePubkey` does not match a provisioned AS on the cluster. The
> marketplace verifies the envelope signature against `X-AS-Pubkey` but does
> **not** tie it to the `agentId` prefix. The vault-api connect check *does*
> enforce `agentId` starts with `agentServicePubkey:` — so use the real
> provisioned AS pubkey, not just any local keypair, for anything that must run.

## Prerequisite B — DDC registry access token

`cef push` writes the manifest + bundle into your Agent Service's **DDC
registry bucket** (the bucket *is* the registry). To authorize the write
without handing out the bucket's raw Sr25519 secret phrase, mint a token in
ROC:

1. Open the **ROC console** → your **Agent Service** → **Access** tab.
2. It shows the AS's registry `bucketId` and a "generate access token" control
   with a selectable validity window.
3. Generate and **copy immediately — the token is shown once.** Note the
   `bucketId` too — you pass it to `cef push --bucket`.

What the token is (from `accountStore.ts` `createRegistryAccessToken`): a
**base58, full-access (GET/PUT/DELETE) DDC AuthToken scoped to that bucket**,
signed by the bucket-owning wallet, with no `subject` — a **bearer** token any
holder can use until it expires. It is **stateless and cannot be individually
revoked**; it only expires (default ~30 days). Prefer a short window and
regenerate on exposure.

Hand it to `cef push` as `--access-token` or `$CEF_DDC_ACCESS_TOKEN`:

```bash
export CEF_DDC_ACCESS_TOKEN="<base58 token from ROC>"
cef push --bucket <bucketId> --as-pubkey <asPubkeyHex>
```

`push.ts` requires **exactly one** of `--access-token` / `$CEF_DDC_ACCESS_TOKEN`
or `--secret-phrase` / `$CEF_DDC_SECRET_PHRASE`. Neither → hard error; both →
hard error. The token path is the common team case (you don't hold the phrase).

## The Deploy action (after `cef publish`)

Once the card is published, go to the agent in the ROC console and click
**Deploy**, selecting the **version** to make live. A deployment pins one
version and makes it live for connected users; the orchestrator only serves a
version a deployment targets.

Under the hood ROC applies a deployment **set** to the orchestrator's
deployments API, keyed by the composite `agentId` (`<asPubkey>:<alias>`). The
minimal "make it live" set is a single default that catches everything:

```jsonc
// PUT /api/v1/agents/<asPubkey>:my-agent/deployments
{
  "author": "you@cere.io",
  "note": "go live",
  "deployments": [
    { "name": "default", "priority": 99, "targeting": "", "weight": 1, "version": "latest" }
  ]
}
```

`validate.go` requires exactly **one default** (a deployment with empty
`targeting`) — the mandatory fallthrough so selection is never empty. It also
requires `weight >= 1`, a non-empty `version` (a semver or the literal
`"latest"`), and unique lowercase slug `name`s. Keep the default at a high
`priority` (e.g. `99`) so more specific, lower-`priority` deployments win when
you add them for a rollout.

To ship a new version: bump `version` in `cef.config.ts`, re-run
build/push/publish, then leave the `latest` default in place, or add a
lower-`priority` (or same-priority, weighted) deployment pinned to the new
semver for a controlled rollout.

## Detect the missing prerequisite → hand back the exact next step

The skill should not just run commands — it should diagnose *which* gate is
unmet and return the precise command or ROC step. Detection cues, in flow order:

| Symptom / state | What's missing | Hand back |
|---|---|---|
| No AS keypair (`~/.cef/*.key` or `~/.config/cef/credentials`, `$CEF_AS_*`) | Publisher identity | `cef keypair generate --out ~/.cef/my-agent.key` (once; share via secrets manager) |
| No AS pubkey to pass `--as-pubkey` | Prerequisite A | "Open ROC → your Agent Service → copy the AS pubkey (hex)" |
| `push` errors "no DDC authorization" | Prerequisite B | "Open ROC → Agent Service → Access → generate token; `export CEF_DDC_ACCESS_TOKEN=…`" (note the `bucketId` for `--bucket`) |
| `push` errors "both … provided" | Ambiguous auth | Unset one of `$CEF_DDC_SECRET_PHRASE` / `$CEF_DDC_ACCESS_TOKEN` (or drop one flag) |
| `push` errors "bundle not found" / "no built agents" / "no version" | Build not run | `cef build` first |
| `publish` errors "bundle not uploaded — run `cef push` first" (empty `bundle.cid`) | Push not run | `cef push --bucket <id> --as-pubkey <hex>` |
| `publish` REPLAY (HTTP 409, nonce reused) | Transient | Re-run `cef publish` (fresh nonce minted each call) |
| Published but a user's `connect` never gets served | No deployment | "Open ROC → the agent → **Deploy** → select version" (or PUT the default set above) |
| `connect` rejected (agentId prefix mismatch) | Wrong AS pubkey | Re-push/publish with the **provisioned** AS pubkey so `agentId` = `<asPubkey>:<alias>` |

Rule of thumb: the CLI's own error strings name the missing gate ("run
`cef build` first", "run `cef push` first", "no DDC authorization"). Surface
that string verbatim, then translate it into the one next command or ROC click
— don't guess further ahead than the current gate.

## The full deploy loop (for reference)

```bash
# one-time: publisher identity (share via secrets manager, not per-dev)
cef keypair generate --out ~/.cef/my-agent.key
# one-time per token window: copy from ROC → Agent Service → Access
export CEF_DDC_ACCESS_TOKEN="<base58 token from ROC>"

cef build
cef push    --bucket <bucketId>            --as-pubkey <asPubkeyHex>
cef publish --keystore ~/.cef/my-agent.key --as-pubkey <asPubkeyHex>
# then, in ROC: click Deploy and select the version   (cef deploy CLI is coming)
```

The marketplace base URL defaults to the dev cluster
(`https://agent-marketplace.compute.dev.ddcdragon.com`); override with
`--marketplace` or `$CEF_MARKETPLACE_URL`. Select the DDC network for `push`
with `--preset MAINNET|TESTNET|DEVNET` (default `MAINNET`) or `--endpoint`.
