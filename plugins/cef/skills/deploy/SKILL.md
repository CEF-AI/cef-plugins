---
name: deploy
description: Build, push, and deploy a CEF agent so it runs live (including in a sandbox); marketplace publish is optional and last. Use when shipping/releasing an agent or when the user asks to build/push/deploy/publish.
---

# Deploy a CEF agent

Ship an agent through this order: **`cef init` → `cef build` → `cef push` → `cef deploy`**.

**`cef deploy` is what makes an agent live and testable** (including in a
sandbox). **`cef publish` — the marketplace listing — is OPTIONAL and the LAST
step.** Publish only places the agent's card in the marketplace for discovery;
skip it entirely for debug, test, or sandbox use. An agent runs because it is
deployed, not because it is published.

In a scaffolded project run `cef` via `npx cef …` (it's a local dev-dep).

```bash
npx cef init my-agent                                 # scaffold (does NOT install deps — fast)
cd my-agent && pnpm install                           # install project deps (incl. the `cef` dev-dep)
npx cef build                                         # local: compile dist/<id>/{bundle.js,manifest.json}
npx cef push    --bucket <id>    --as-pubkey <hex>    # OUTWARD: upload bundle to the DDC registry
npx cef deploy  --endpoint <url> --as-pubkey <hex>    # OUTWARD: apply deployments/ — makes the agent live
npx cef publish --keystore <path> --as-pubkey <hex>   # OPTIONAL, LAST: list the card in the marketplace
```

## Non-negotiable rules

- **`push`, `deploy`, and `publish` are outward-facing.** Never run any of them
  without confirming with the user first (see checklist below). `build` and
  `init` are local and safe — run them freely.
- **Two setup steps remain human-only, done in the ROC console:** creating the
  Agent Service (which yields the AS pubkey) and minting a DDC registry access
  token. Hand those back as steps; never fake them. The ROC **Deploy** button
  does the same thing as `cef deploy`.
- **Never invent flags or commands.** Everything here is grounded in the
  references.

## The `deployments/` folder is the source of truth

An agent project declares its deployment records as files — one record per
`deployments/<name>.jsonc`, the filename stem being the record `name` (the same
convention as `migrations/<cubby>/` and `widgets/<id>/`). `cef deploy` reads the
whole folder, assembles the deployment SET, and applies it.

A record's key fields: `priority` (int, REQUIRED, **lower wins**), `targeting`
(string, REQUIRED — a CEL expression over `vault.id` / `vault.scope` /
`event.type`; `""` means always eligible), `version` (REQUIRED — a semver or the
literal `"latest"`), and `weight` (int ≥ 1, default 1, splits traffic within a
priority tier for A/B). Full field list and SET rules:
[references/roc-deploy.md](./references/roc-deploy.md).

The folder must contain **exactly one** record with empty `targeting` — the
mandatory default/fallthrough. Give it a HIGH priority number (e.g. 99) so more
specific, lower-priority records win. `version: "latest"` fails closed if
`latest` was never pushed — pin a semver for anything that must run.

## `cef deploy` — the command surface

```bash
npx cef deploy [--version <v>] [--endpoint <url>] \
               [--as-pubkey <hex>] [--author <who>] [--note <msg>] [--dry-run]
```

- Reads `deployments/`; **errors if the folder is absent or empty**.
- `--endpoint` (or `$CEF_ENDPOINT`) selects WHERE to apply, defaulting to the
  dev environment. It is **orthogonal** to the files: the same `deployments/`
  folder applies to any environment, so never put environment names inside the
  files.
- `--version` overrides the record `version` without editing files — the
  everyday "pushed a new build, roll it out" step.
- `--deployments <path>` points at the records: a directory (default
  `deployments/`, one record per file) or a single set/record file — like
  `kubectl apply -f <file|dir>`.
- The whole set is applied at once (declarative desired state) — records removed
  locally disappear remotely on the next deploy.
- `--dry-run` shows what would be applied without applying it.

## Before push / deploy / publish — confirm identity, bucket, target

Surface these to the user and get an explicit go-ahead before any outward call:

1. **AS pubkey** (`--as-pubkey <hex>`) — the Agent Service routing identity from
   ROC. Same value on push, deploy, and publish. A wrong pubkey means the agent
   is unroutable (`connect` rejects on an agentId prefix mismatch).
2. **Bucket** (`--bucket <numericId>`, push only) — the AS's DDC registry bucket,
   in decimal.
3. **DDC auth** (push only) — exactly one of `--access-token` /
   `$CEF_DDC_ACCESS_TOKEN` (ROC-minted, the common team case) or
   `--secret-phrase` / `$CEF_DDC_SECRET_PHRASE` (bucket owner). Both = error;
   neither = error.
4. **Endpoint** (deploy) — echo the resolved `--endpoint` / `$CEF_ENDPOINT` so
   the user knows which environment goes live.
5. **AS keypair** (publish) — resolved via `--privkey/--pubkey`, `CEF_AS_*` env,
   `--keystore`, or a `~/.config/cef/credentials` profile.

Echo the resolved AS pubkey, bucket, endpoint, and (for publish) marketplace
URL, then ask to proceed. See
[references/credentials.md](./references/credentials.md) for the two-identity
model and resolution order.

## Detect the missing prerequisite → hand back the exact next step

Diagnose *which* gate is unmet and return the one precise command or ROC click.
The CLI's own error strings name the gate — surface them verbatim, then act.

| Symptom / state | Missing | Hand back |
|---|---|---|
| No AS keypair anywhere (`CEF_AS_*`, keystore, credentials profile) | Publisher identity | `cef keypair generate --out ~/.cef/my-agent.key` (once; share via a secrets manager, not per-dev) |
| No AS pubkey to pass `--as-pubkey` | ROC setup step 1 | "Open ROC → your Agent Service → copy the AS pubkey (hex)" |
| `push`: "no DDC authorization" | ROC setup step 2 | "ROC → Agent Service → Access → generate token; `export CEF_DDC_ACCESS_TOKEN=…`" (note the `bucketId`) |
| `push`: "both … provided" | Ambiguous auth | Unset one of `$CEF_DDC_SECRET_PHRASE` / `$CEF_DDC_ACCESS_TOKEN` |
| `push`: bundle/version not found | Build not run | `cef build` first |
| `push`: missing `@cere-ddc-sdk/ddc-client` | Optional dep | `pnpm add @cere-ddc-sdk/ddc-client` |
| `deploy`: "no deployments/ folder" / empty folder | No records | Create `deployments/default.jsonc` (see reference), then re-run |
| Pushed, but a user's `connect` never gets served | Not deployed | `npx cef deploy --endpoint <url> --as-pubkey <hex>` (or ROC → the agent → **Deploy**) |
| `connect` rejected (agentId prefix mismatch) | Wrong AS pubkey | Re-push/deploy with the **provisioned** AS pubkey |
| Card missing from the marketplace (agent still runs) | Not published | `npx cef publish …` — optional, only for discovery |

Rule of thumb: don't guess past the current gate — resolve it, then re-run.

## Shipping a new version

Bump `version` in `cef.config.ts`, then re-run build → push → `cef deploy`
(pass `--version <semver>` to roll the new build out without editing files, or
deploy `latest`). Add a lower-priority or weighted record in `deployments/` for
a controlled rollout. Re-run `cef publish` only if the marketplace card needs
refreshing — it is not required for the new version to run.
