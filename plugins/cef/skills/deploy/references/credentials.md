# Credentials, keypair & DDC bucket

Two independent identities are involved in shipping an agent, and they are
easy to confuse:

| Command | Identity | Key type | Where it goes |
| --- | --- | --- | --- |
| `cef push` | **DDC bucket owner** (or a token from them) | Sr25519 (substrate) | authorizes writes to the content-addressed bucket |
| `cef push --vault` | **Your own wallet key** (or a wallet-api token) | Sr25519 or Ed25519 | authorizes vault-api to publish into a vault scope you can write |
| `cef publish` | **Agent Service (AS)** | Ed25519 | signs the marketplace publish envelope |

They are unrelated keys. `cef push` signs DDC writes with the bucket-owner
key, which "can be anything with write access" — so the AS identity **cannot
be derived** from it and must be passed in explicitly with `--as-pubkey`.

## AS Ed25519 keypair (`cef publish`, and `--as-pubkey` on push)

The AS keypair is the **Agent Service identity** — the marketplace-visible
publisher of the agent — and is **typically shared across a team**, not
unique per developer. The CLI never auto-generates it; you provision it
once with `cef keypair generate` and share it through your secrets channel.
Generating a fresh one per developer fragments the agent's marketplace
history under different identities.

### Generate a keypair

`cef keypair generate` is the only sanctioned path to a new AS keypair:

```bash
# JSON to stdout — pipe to a secrets manager
cef keypair generate

# Write a 0600 file (refuses an existing file unless --force)
cef keypair generate --out keypair.json --force

# Merge into the shared credentials file under [staging]
# (atomic 0600 write; refuses an existing section unless --force)
cef keypair generate --profile staging
```

Keypair shape: `pubkey` is 64 lowercase hex chars (32 bytes); the private
key is a 32-byte Ed25519 **seed**, also 64 hex chars.

### Resolution order

`cef publish` resolves the AS keypair AWS-CLI-style — flags beat env, env
beats files, files beat the shared INI. The **first** source that yields a
complete keypair wins:

| # | Source | How |
| --- | --- | --- |
| 1 | CLI flags | `--privkey <hex> --pubkey <hex>` |
| 2 | Env vars | `CEF_AS_PRIVATE_KEY=<hex> CEF_AS_PUBLIC_KEY=<hex>` |
| 3 | Keystore via flag | `--keystore <path>` — JSON `{ pubkey, privkey }` |
| 4 | Keystore via env | `CEF_AS_KEYSTORE=<path>` — same shape |
| 5 | Profile in shared file | `~/.config/cef/credentials`, section via `--profile <name>` / `CEF_PROFILE`, default `default` |

Gotchas:

- **Pairs must be complete.** Setting only one of `--privkey`/`--pubkey`
  (or only one of the two env vars) is a hard error, not a fall-through.
- **A named path or profile must exist.** `--keystore <path>`,
  `CEF_AS_KEYSTORE`, and an explicit `--profile <name>` all error if the
  file/section is missing — the miss is not silently skipped (that would
  mask a typo). Only a *default* profile with no file present counts as
  "no credentials".
- With no source at all, the CLI prints multi-line guidance listing every
  source and pointing at `cef keypair generate`, then exits non-zero.

### Shared credentials file

Lives at `~/.config/cef/credentials` (or `$XDG_CONFIG_HOME/cef/credentials`),
mode `0600`, minimal INI. Multiple profiles hold identities for several
environments on one workstation:

```ini
[default]
pubkey = abc...
privkey = abc...

[staging]
pubkey = def...
privkey = def...
```

## DDC bucket + credentials (`cef push`)

`cef push` uploads `dist/<agentId>/bundle.js` to DDC as a content-addressed
DAG and rewrites `manifest.json` so `bundle` is `{ bucketId, cid }` instead
of embedded inline. It requires:

1. **`--bucket <numericId>`** (REQUIRED) — decimal DDC bucket id. The
   substrate signer must own it. The bucket *is* the agent registry
   (`CNS "agents" → directory → per-agent record → per-version node`).
2. **Exactly one** DDC write authorization:
   - `--secret-phrase <phrase>` / `$CEF_DDC_SECRET_PHRASE` — the Sr25519
     phrase that **owns** the bucket.
   - `--access-token <token>` / `$CEF_DDC_ACCESS_TOKEN` — a base58 token
     **granted by the bucket owner** (e.g. minted in ROC by the wallet that
     owns the AS's bucket). Use this when you don't hold the bucket's seed;
     the client is built with a throwaway signer and the token carries the
     authority. Providing **both** phrase and token is an error; providing
     **neither** is an error.
3. **`--as-pubkey <hex>`** — the AS pubkey (exactly as shown in ROC).
   Because push's signing key is unrelated to the AS identity, this is how
   the manifest gets stamped: `agentServicePubkey = <asPubkey>` and
   `agentId = <asPubkey>:<alias>`. With it stamped, bucket consumers (the
   dashboard registry read, the `connect` check) see a fully-identified
   manifest immediately — no need to wait for `cef publish`.

```bash
# Owner pushes with the bucket's secret phrase
cef push --bucket 1234 \
  --secret-phrase "$CEF_DDC_SECRET_PHRASE" \
  --as-pubkey <hex>

# Teammate pushes with a ROC-minted access token (no bucket seed)
cef push --bucket 1234 \
  --access-token "$CEF_DDC_ACCESS_TOKEN" \
  --as-pubkey <hex> --preset TESTNET
```

Other push flags: `--preset MAINNET|TESTNET|DEVNET` (default MAINNET),
`--endpoint <url>` (overrides preset), `--agent <id>` (when `dist/` holds
more than one), `--out <dir>` (default `dist`).

Gotchas:

- Run `cef build` first — push errors if `dist/<id>/{bundle.js,manifest.json}`
  or the manifest `version` is missing.
- `cef push` needs the optional dep `@cere-ddc-sdk/ddc-client`; install it
  (`pnpm add @cere-ddc-sdk/ddc-client`) or push throws an actionable error.
- `bucketId` is recorded in the manifest as a **decimal string**.
- On a bucket's first push the `agents` CNS root doesn't exist yet; push
  logs that it's creating the registry root (the internal resolve miss is
  expected, not a failure). Set `CEF_DEBUG` for full ddc-client logs.

## Org vault publish (`cef push --vault`)

For an agent that belongs to an **organization vault**, push through
vault-api instead of writing DDC yourself. vault-api checks that the caller
holds write on the scope (the vault owner, or a member with a member-level
grant on it), writes the vault's registry bucket, and binds the agent's alias
to the scope it is first published from, so another scope cannot replace it.

```bash
# The user has exported ONE of these in their own shell — never ask for the value:
#   CEF_VAULT_SECRET_PHRASE  (their wallet key's phrase)   or   CEF_VAULT_TOKEN  (wallet-api token)
cef push --vault <vaultId> --vault-scope <scope> \
  --as-pubkey 0x<agent-service-pubkey> --env dev
```

Flags:

- `--vault <vaultId>` + `--vault-scope <scope>` (both REQUIRED for this path).
  `--vault-scope`, not `--scope` — `--scope` belongs to `--kind external`.
- Exactly one credential: `--secret-phrase <phrase>` / `$CEF_VAULT_SECRET_PHRASE`
  (the user's own wallet key, always signed as ed25519)
  or `--vault-token <token>` / `$CEF_VAULT_TOKEN` (wallet-api bearer token).
  Both or neither is an error. **`$CEF_DDC_SECRET_PHRASE` is never read here**
  — it names a bucket owner, not the developer.
- `--as-pubkey 0x<hex>` — stamps `agentId` / `agentServicePubkey` as on the
  bucket path.
- `--env dev|stage|prod` picks the vault-api endpoint; `--vault-api <url>`
  overrides it.
- `--agent <id>`, `--out <dir>` — as on the bucket path.
- Refused with `--vault`: `--bucket`, `--access-token`, `--preset`,
  `--endpoint`, `--cdn`, `--subject-phrase`. Refused without it: `--vault-scope`, `--vault-api`,
  `--vault-token`. All vault flags are refused under
  `--kind external`.

The CLI uploads the version's files with the manifest **unstamped** (no
`bundle` / widget `dag`); vault-api stamps the CIDs and returns the stored
manifest, which `cef push` writes back to `dist/<id>/manifest.json`.

Errors (the server's own code and message are appended to each):

| Code / status | Meaning | What to do |
| --- | --- | --- |
| `403 ALIAS_NOT_YOURS` | The alias is bound to another scope (or is already in the registry unclaimed, which only the vault owner can claim) | Push from the alias's own scope, or rename the agent |
| other `403` | No write on the scope, or the scope doesn't exist | Ask the vault owner for a member-level grant, or fix the scope name |
| `409 REGISTRY_NOT_CONFIGURED` | The org has not enabled agent publishing (no registry bucket/delegation recorded, or it expired) | Only the org owner can fix it: ROC → the organization page → **Enable agent publishing**; then push again |
| `503 REGISTRY_UNAVAILABLE` | Registry temporarily unreadable/unwritable; nothing changed | Retry |
| `401` | Credential rejected | Check phrase/token and clock skew (within 5 minutes) |
| `404 VAULT_NOT_FOUND` | No such vault on this vault-api | Check `--vault` and `--env` |
| `404` (no code) | This vault-api doesn't serve the publish route | Check `--vault-api` / `--env` |

The direct-bucket path above is unchanged and remains the way to publish into
a bucket that is not an org vault's registry (e.g. a personal Agent Service
bucket).
