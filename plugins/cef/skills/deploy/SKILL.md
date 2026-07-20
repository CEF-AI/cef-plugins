---
name: deploy
description: Build, push, publish, and deploy a CEF agent to the platform. Use when shipping/releasing an agent, or when the user asks to build/push/publish/deploy.
---

# Deploy a CEF agent

Ship an agent through four stages: **build → push → publish → deploy**.

In a scaffolded project run `cef` via `npx cef …` (it's a local dev-dep).

```bash
npx cef build                                          # local: compile dist/<id>/{bundle.js,manifest.json}
npx cef push    --bucket <id>      --as-pubkey <hex>   # OUTWARD: upload bundle to DDC registry
npx cef publish --keystore <path>  --as-pubkey <hex>   # OUTWARD: sign + publish marketplace card
npx cef deploy  --endpoint <url>   --as-pubkey <hex>   # OUTWARD: pin a version live
```

Stages 1–3 write to DDC / the marketplace; stage 4 (`cef deploy`) applies a
deployment via the platform API — the ROC **Deploy** button does the same.
`--endpoint` defaults to dev (`$CEF_ENDPOINT` to override; test = swap
`dev`→`test`). **Creating the Agent Service and minting the DDC token remain
human actions in ROC.**

## Non-negotiable rules

- **`push` and `publish` are outward-facing and hard to reverse.** Never run
  either without confirming with the user first (see checklist below). `build`
  is local and safe — run it freely.
- **`cef deploy` is fine to run — but confirm first, like push/publish.** What
  stays human-only is **creating the Agent Service and minting the DDC token**
  in ROC; hand those back as steps, never fake them.
- **Never invent flags or commands.** Only `build`, `push`, `publish`, `deploy`,
  `keypair`, `inspect` exist. Everything here is grounded in the references.

## Before push / publish — confirm identity, bucket, marketplace

Surface these to the user and get an explicit go-ahead before the outward call:

1. **AS pubkey** (`--as-pubkey <hex>`) — the provisioned Agent Service routing
   identity from ROC. Same value on both `push` and `publish`. Wrong pubkey =
   published but unroutable (`connect` rejects agentId prefix mismatch).
2. **Bucket** (`--bucket <numericId>`) — the AS's DDC registry bucket. Decimal.
3. **DDC auth** — exactly one of `--access-token`/`$CEF_DDC_ACCESS_TOKEN`
   (ROC-minted, common team case) or `--secret-phrase`/`$CEF_DDC_SECRET_PHRASE`
   (bucket owner). Both = error; neither = error.
4. **AS keypair** for publish — resolved via `--privkey/--pubkey`, env
   `CEF_AS_*`, `--keystore`, or `~/.config/cef/credentials` profile.
5. **Marketplace / network** — publish defaults to the dev cluster
   (`--marketplace`/`$CEF_MARKETPLACE_URL`); push network is
   `--preset MAINNET|TESTNET|DEVNET` (default MAINNET) or `--endpoint`.

Echo the resolved AS pubkey, bucket, target network, and marketplace URL, then
ask to proceed. See [references/credentials.md](./references/credentials.md)
for the two-identity model and resolution order.

## Detect the missing prerequisite → hand back the exact next step

Diagnose *which* gate is unmet and return the one precise command or ROC click.
The CLI's own error strings name the gate — surface them verbatim, then act.

| Symptom / state | Missing | Hand back |
|---|---|---|
| No AS keypair anywhere (`CEF_AS_*`, keystore, credentials profile) | Publisher identity | `cef keypair generate --out ~/.cef/my-agent.key` (once; share via secrets manager, not per-dev) |
| No AS pubkey to pass `--as-pubkey` | ROC Prereq A | "Open ROC → your Agent Service → copy the AS pubkey (hex)" |
| `push`: "no DDC authorization" | ROC Prereq B | "ROC → Agent Service → Access → generate token; `export CEF_DDC_ACCESS_TOKEN=…`" (note the `bucketId`) |
| `push`: "both … provided" | Ambiguous auth | Unset one of `$CEF_DDC_SECRET_PHRASE` / `$CEF_DDC_ACCESS_TOKEN` |
| `push`: bundle/version not found | Build not run | `cef build` first |
| `push`: missing `@cere-ddc-sdk/ddc-client` | Optional dep | `pnpm add @cere-ddc-sdk/ddc-client` |
| `publish`: "run `cef push` first" (empty `bundle.cid`) | Push not run | `cef push --bucket <id> --as-pubkey <hex>` |
| `publish`: REPLAY / HTTP 409 | Transient nonce reuse | Re-run `cef publish` |
| Published, but a user's `connect` never gets served | No deployment | `npx cef deploy --endpoint <url> --as-pubkey <hex>` (or ROC → the agent → **Deploy**) |
| `connect` rejected (agentId prefix mismatch) | Wrong AS pubkey | Re-push/publish with the **provisioned** AS pubkey |

Rule of thumb: don't guess past the current gate — resolve it, then re-run.

## The two ROC touchpoints (human-only)

- **Before push** — create the Agent Service (yields the AS pubkey) and mint a
  DDC registry access token. Walk the human through it.
- **After publish** — run `npx cef deploy` (or click **Deploy** in ROC) to make
  a version live; the platform serves only a version a deployment targets
  (ADR-038). `cef deploy` needs network reachability to the `--endpoint`.

Full ROC walkthrough, the deployment set shape, and go-live rules:
[references/roc-deploy.md](./references/roc-deploy.md).

## Shipping a new version

Bump `version` in `cef.config.ts`, re-run build → push → publish → `cef deploy`
(pass `--version <semver>` to pin the new build, or deploy `latest`); or adjust
the deployment in ROC for a controlled rollout.
