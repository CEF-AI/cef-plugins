# Workflow releases: `cef release` → `cef deploy <version>`

For a **workflow** (an agent whose manifest carries a graph), ship with
`cef build` → `cef release` → `cef deploy <version>`. `cef push` still works
but publishes an unchecked version with no release record and moves `latest`;
prefer `cef release` + `cef deploy <version>`.

| Step | Creates a version | Runs checks | Moves the deployment | Moves `latest` |
|---|---|---|---|---|
| `cef push` | yes | no | no | **yes** |
| `cef release` | yes (only if required datasets pass) | yes | no | no |
| `cef deploy <version>` | no | no | yes (default deployment) | **yes** |

## Why `latest` matters

vault-api delivers a workflow's events, schedules and webhooks from the
manifest `latest` names. So `latest` = the deployed version: a released but
undeployed version must not move it, and `cef deploy <version>` (or a
rollback) does.

## `cef release`

```bash
npx cef release [version] [--title <text>] [--notes <text> | --notes-file <file>] [--skip-review] [--json]
```

1. Hashes the built graph in `dist/`; reuses results already cached for that
   content hash.
2. Otherwise publishes a check candidate `0.0.0-c.<hash7>` (never `latest`) and
   runs against it:
   - the **AI review** — never blocks; `--skip-review` skips it (a review
     already cached for this graph is kept);
   - every dataset marked **Required for release** — blocks; cannot be skipped.
3. A required dataset fails → no version is created, **exit 1**.
4. Otherwise publishes the numbered version with its `release` block. Not
   deployed; `latest` does not move.

Defaults: `version` = the patch after the newest release; `--title` = the last
commit's subject; `--notes` = commit subjects since the previous release.
`--notes` and `--notes-file` are mutually exclusive.

**Publish destination** — same meanings as `cef push`:

- DDC bucket: `--bucket <id>` (default: the built manifest's bundle bucket) +
  exactly one of `--secret-phrase` / `$CEF_DDC_SECRET_PHRASE` or
  `--access-token` / `$CEF_DDC_ACCESS_TOKEN` (`--subject-phrase` /
  `$CEF_DDC_SUBJECT_PHRASE` for a subject-bound token); `--preset`,
  `--endpoint`, `--cdn`.
- Org vault: `--vault <vaultId> --vault-scope <scope>` + `$CEF_VAULT_TOKEN`
  (`--vault-token`) or `$CEF_VAULT_SECRET_PHRASE` (`--secret-phrase`).

**What the checks need:**

| Flag | Env | For |
|---|---|---|
| `--cli-token` | `$CEF_ACCESS_TOKEN` | orchestrator + vault-api |
| `--keystore` + `--password` / `--wallet-phrase` | `$CEF_KEYSTORE` + `$CEF_KEYSTORE_PASSWORD` / `$CEF_SECRET_PHRASE` | ed25519 wallet for vault-api |
| `--evals-vault <id>` | `$CEF_VAULT_ID` | vault the workflow runs in (default: the wallet's own) |
| `--scope <scope>` | | vault scope (default: `requiredScopes`, else `default`) |
| `--inference-key` | `$CEF_INFERENCE_KEY` | Inference Gateway key the AI review is billed to |
| `--review-model` | `$CEF_REVIEW_MODEL` | review model `bucket/name/version` |
| `--as-pubkey`, `--agent`, `--out`, `--env`, `--orchestrator`, `--vault-api`, `--inference-gateway`, `--timeout <ms>` | | overrides |

### Where "Required for release" is set

ROC › the workflow › **Evaluations** › the dataset heading. Rule per dataset:
**not worse than live**, or **at least N%** accuracy. With no dataset marked
required, a release is only the AI review and a number.

## `cef deploy <version>`

```bash
npx cef deploy <version> [--accept-warnings] [--dry-run] [--json] [--note <text>] [--author <who>]
```

- Shows the release's marks, repoints the **default** deployment at
  `<version>`, then moves `latest` to it.
- **Rollback** = `cef deploy <older-version>`.
- AI review warnings: a terminal asks; CI needs `--accept-warnings`.
- `--dry-run`: prints what would change (deployment and/or `latest`), changes
  nothing.
- Already live and `latest` already names it → no-op.
- Versions published with `cef push` have no release block ("Not checked") and
  stay deployable.

**Moving `latest`** — where the workflow is published:

- DDC: `--secret-phrase` / `$CEF_DDC_SECRET_PHRASE` or `--ddc-access-token` /
  `$CEF_DDC_ACCESS_TOKEN` (`--subject-phrase` for a subject-bound token);
  `--bucket`, `--cdn`, `--preset`, `--ddc-endpoint`.
- Org vault: `--vault <vaultId> --vault-scope <scope>` + `$CEF_VAULT_TOKEN`
  (`--vault-token`) or `$CEF_VAULT_SECRET_PHRASE` (`--secret-phrase`);
  `--vault-api`.

> On `cef deploy`, `--access-token` is the **CLI access token** for the
> deployment API (`$CEF_ACCESS_TOKEN`) and `--endpoint` is the **platform API**.
> The DDC equivalents are `--ddc-access-token` and `--ddc-endpoint`.

Without `<version>`, `cef deploy` applies the `deployments/` folder as before
([roc-deploy.md](./roc-deploy.md)).

## Reading releases

```bash
npx cef release list [--json]               # newest first, marks, what is live
npx cef release show <version> [--json]     # author, notes, AI review, experiments
npx cef release diff <from> <to> [--json]   # steps added, removed, changed
npx cef check <version> [--skip-review] [--json]   # re-run checks on an existing release; exit 1 on failure; release unchanged
npx cef eval cases <experiment> [--failed] [--json]  # an experiment's runs case by case
```

`list`/`show`/`diff` also take `--env`, `--agent`, `--out`, `--as-pubkey`,
`--bucket`, `--cdn`, `--orchestrator`, `--cli-token`. `cef check` takes the
same check flags as `cef release`.

## Errors

| Symptom | Hand back |
|---|---|
| `release`: "Not released. N required dataset(s) failed." (exit 1) | `cef eval cases <experiment> --failed`; fix, `cef build`, release again |
| "which bucket is the workflow in?" | Pass `--bucket <id>` (or push once so the manifest names one) |
| "--as-pubkey is required to run experiments" / "no Agent Service pubkey" | Pass `--as-pubkey <hex>` |
| "--vault needs --vault-scope" | Add `--vault-scope <scope>` |
| "give --notes or --notes-file, not both" | Drop one |
| `deploy`: "<v> has N review warnings. Re-run with --accept-warnings." | No terminal to ask (CI): confirm with the user, then add `--accept-warnings` |
| `deploy`: "<v> is deployed …, but `latest` still names …" | Moving `latest` failed (credential?): fix it, re-run `cef deploy <v>` |
| `push` prints "is a workflow: this published an unchecked version" | Informational; prefer `cef release` + `cef deploy <version>` |
