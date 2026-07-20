---
name: deploy
description: Build, push, publish, and deploy a CEF agent to the platform. Use when shipping/releasing an agent, or when the user asks to build/push/publish/deploy.
---

# Deploy a CEF agent

Ship an agent through four stages: **build → push → publish → deploy**.

```bash
cef build                                          # local: compile dist/<id>/{bundle.js,manifest.json}
cef push    --bucket <id>      --as-pubkey <hex>   # OUTWARD: upload bundle to DDC registry
cef publish --keystore <path>  --as-pubkey <hex>   # OUTWARD: sign + publish marketplace card
# then a HUMAN in ROC clicks Deploy and selects the version (no cef deploy CLI yet)
```

Stages 1–2 write to the shared DDC bucket; 3 publishes to the marketplace; 4
and creating the Agent Service / minting tokens are **human actions in ROC**.

## Non-negotiable rules

- **`push` and `publish` are outward-facing and hard to reverse.** Never run
  either without confirming with the user first (see checklist below). `build`
  is local and safe — run it freely.
- **Do not run the Deploy step, create Agent Services, or mint tokens.** Those
  are done by a human in the ROC console. Hand back the exact command/click.
- **Never invent flags or commands.** Only `build`, `push`, `publish`,
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
| Published, but a user's `connect` never gets served | No deployment | "Open ROC → the agent → **Deploy** → select version" |
| `connect` rejected (agentId prefix mismatch) | Wrong AS pubkey | Re-push/publish with the **provisioned** AS pubkey |

Rule of thumb: don't guess past the current gate — resolve it, then re-run.

## The two ROC touchpoints (human-only)

- **Before push** — create the Agent Service (yields the AS pubkey) and mint a
  DDC registry access token. Walk the human through it.
- **After publish** — click **Deploy** and select the version to make live;
  the orchestrator serves only a version a deployment targets (ADR-038).

Full ROC walkthrough, the deployment set shape, and go-live rules:
[references/roc-deploy.md](./references/roc-deploy.md).

## Shipping a new version

Bump `version` in `cef.config.ts`, re-run build → push → publish, then in ROC
either keep the `latest` default or pin the new semver for a controlled rollout.
