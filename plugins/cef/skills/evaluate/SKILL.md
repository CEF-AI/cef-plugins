---
name: evaluate
description: Evaluate a CEF workflow with `cef eval` — keep a dataset of cases in the repo, sync it with the agent service's bucket, run it as an experiment against a workflow version, and compare experiments. Use when the user asks to test, score, benchmark, regression-check, or compare workflow versions, or mentions datasets, cases, experiments, baselines, or `cef eval`.
---

# Evaluate a CEF workflow

`cef eval` runs a **dataset** (cases: a `workflow.start` payload and the Result it
should produce) against a **workflow version**, scores each run's Result field by
field, records duration / tokens / cost / steps, and compares the experiment with
a **baseline**. Warn-only: nothing here blocks a deploy. ROC's **Evaluations**
page reads and writes the same storage, so either side sees the other's work.

Every command is about one workflow: `--workflow <id>` (the workflow agent's
alias), by default the one `cef.config.ts` (or `dist/`) in the current directory
declares. Run the commands from the workflow's folder. In a scaffolded project
run `cef` via `npx cef …`.

```bash
npx cef eval datasets                                  # the workflow's datasets
npx cef eval create <dataset> [--description …] [--max-duration-ms N] [--max-tokens N] [--max-steps N] [--max-cost X]
npx cef eval pull <dataset> [--dir datasets] [--force] # bucket → repo dir
npx cef eval push <dataset> [--dir datasets] [--cut [note]] [--force] [--prune] [--restore]   # repo dir → bucket
npx cef eval run <dataset> [--workflow-version v] [--dataset-version N] [--repeats N] [--concurrency N]
                           [--timeout ms] [--baseline <exp>] [--resume <exp>] [--note …]
npx cef eval experiments [dataset]                     # experiment history (★ = baseline)
npx cef eval compare <base-exp> <candidate-exp>        # how the candidate differs
npx cef eval migrate                                   # once, for a bucket written by the first release
```

Every command takes `--json`. `cef eval run --json` prints
`{ experiment, result, comparison? }` — `result` is the Result check below.

## Non-negotiable rules

- **`push`, `run`, `create`, and `migrate` are outward-facing.** `push`/`create`/
  `migrate` write the agent service's bucket; `run` starts real workflow runs in
  a vault (model calls cost money) and, for a non-live version, adds a
  temporary deployment row. Confirm with the user before each. `datasets`,
  `pull`, `experiments`, and `compare` only read the bucket (`pull` writes the
  local repo dir).
- **Never pass `--force`, `--prune --force`, or `--restore` on your own.** Each
  overrides a guard that protects someone's edit; surface the CLI's refusal
  verbatim and let the user choose.
- **Never ask for, accept, or echo a secret.** Tell the user which env var to
  set in their own shell and report only whether it is set.
- **Never invent flags.** In particular, use `--dataset-version` and
  `--workflow-version` — `--version` is the CLI's own version flag.

## Credentials — what every command needs

| Need | Flag | Env var |
|---|---|---|
| Agent service (names the dataset bucket `as-<pubkey>`) | `--as-pubkey <hex>` | `$CEF_AS_PUBKEY` |
| Bucket credential, either: the owner's ed25519 wallet | `--keystore <file>` + `--password`, or `--secret-phrase` | `$CEF_KEYSTORE` + `$CEF_KEYSTORE_PASSWORD`, or `$CEF_SECRET_PHRASE` |
| …or an S3 key pair issued for that wallet (both halves) | `--access-key` + `--secret` (+ `--bucket` if no `--as-pubkey`) | `$CEF_EVAL_KEY` + `$CEF_EVAL_SECRET` |
| Orchestrator (`run` only: deployments, jobs) | `--access-token` | `$CEF_ACCESS_TOKEN` (ROC → Access, the same token `cef deploy` uses) |
| Environment | `--env dev\|stage\|prod` | `$CEF_ENV` |

- `$CEF_DDC_ACCESS_TOKEN` (what `cef push` uses) **cannot** open dataset storage
  — it is limited to the registry bucket and cannot be delegated to the S3
  gateway. With only that token set, `cef eval` says so; hand back the wallet or
  key-pair option above.
- Giving only one of `--access-key` / `--secret` is an error naming the missing half.
- A wallet's S3 credential is minted once and cached for its hour in
  `~/.cef/eval-credentials.json` (owner-only; `CEF_EVAL_CREDENTIAL_CACHE=<file>`
  moves it, `=off` disables it). The gateway caps live credentials per wallet,
  so do not disable the cache to "fix" an auth error.
- `run` also needs the vault: `--vault <id>` / `$CEF_VAULT_ID`, default the
  wallet's own; `--scope` (default `default`) is the scope runs are published into.

## The repo dir

```
datasets/<dataset>/header.json          Header: dataset, workflowId, description, limits (no baseline — that stays on the platform)
datasets/<dataset>/cases/<caseId>.json  each case at its latest revision; file name = <id>.json
datasets/<dataset>/.versions.json       what the last pull/push synced: each case's revision and content hash, versions cut
```

Commit `header.json` and `cases/`; `.versions.json` is per-checkout sync state.

A case:

```json
{
  "id": "refund-angry",
  "input": { "text": "I want my money back NOW" },
  "expected": {
    "category": "billing",
    "confidence": ">= 0.8",
    "tags": { "$contains": "refund" },
    "reply.language": { "$oneOf": ["en", "en-GB"] }
  },
  "limits": { "maxDurationMs": 20000 }
}
```

- `input` is the `workflow.start` payload. `expected` is keyed by Result field;
  dot paths reach into nested fields. A field the output lacks **fails**.
- A plain value is deep equality (an object is a subset match); a string like
  `">= 0.8"` is a numeric band against a number. Matchers: `$eq $oneOf
  $contains $regex $between $gte $lte $exists $approx`+`$tol`. A matcher with
  several operators needs **all** of them (`{"$exists": true, "$gte": 1}`).
  `$between` is `[min, max]`, min ≤ max.
- `$regex` patterns over 256 characters, or that quantify a group containing a
  quantifier (`(a+)+`, `(\w+\s?)*`), are refused (the field fails with
  `unsafe $regex refused`) — they can hang scoring. Write anchored, flat patterns.
- Limits (`maxDurationMs maxTokens maxSteps maxCost`, header-wide or per case)
  only **warn** (counted as breaches); they never fail a case.

The scored output is the workflow's declared **Result** — its `output` node's
`params.result` (and `resultTypes`). `run` reads the version's Result (the live
deployment's graph when the version is live, else `cef.config.ts`/`dist/` of
exactly that version) and warns before starting about cases expecting a field
the version does not return, or returns with another type; those score **n/a**,
not fail. If the user's expectations are all n/a, the fix is the dataset or the
Result declaration, not the workflow logic.

## The loop

1. **Dataset.** `cef eval create <dataset>`, or write `datasets/<dataset>/header.json`
   + cases by hand and `cef eval push <dataset>` (creates it). To start from the
   bucket's copy (e.g. cases a teammate added in ROC): `cef eval pull <dataset>`.
2. **Edit cases in the repo**, then `cef eval push <dataset>`. Only changed cases
   get a new revision; nothing is rewritten in place.
3. **Run.** `cef eval run <dataset>` runs the latest dataset version (cutting a
   new one first if the cases changed) against the live workflow version.
   `--workflow-version <v>` runs a version that is pushed but not live: it is
   pinned for this experiment's runs only, then released. `--repeats 3` runs
   each case three times (for nondeterministic model output).
4. **Compare.** The run compares itself with `--baseline <exp>`, else the
   dataset's baseline (chosen in ROC). `cef eval compare <a> <b>` compares any
   two. When the dataset changed between them, only cases unchanged on both
   sides are compared (`comparedOn`), and the output says what changed — an
   edited expectation is not evidence of a regression.
5. **Resume** an experiment that died: `cef eval run <dataset> --resume <exp>`
   with the same `--workflow-version` / `--repeats`. Runs already recorded are
   reused; the rest run in fresh contexts.

## Refusals → what to hand back

All of these refuse **before writing anything**. Quote the CLI's message, then:

| Refusal | Meaning | Options for the user |
|---|---|---|
| `pull`: "case files were edited since the last pull/push and differ from the bucket" | Unpushed local edits would be overwritten (or a locally edited case was removed in the bucket) | `cef eval push` them first; or move the edits aside; or `pull --force` to take the bucket's version |
| `push`: "changed in the bucket since they were pulled, and differ from the repo" | Someone edited the case in ROC since your pull | `pull`, redo the edit, push; or `push --force` to write the repo's version over theirs |
| `push --prune`: "would delete these cases, edited in the bucket since they were pulled" | You removed a case locally that someone else edited | `pull` to get it back; or `push --prune --force` to delete it, edit and all |
| `push`: "deleted or archived in the bucket, and still in the repo dir" | Someone removed it in ROC | `pull` to drop it locally; give it a new id; or `push --restore` to bring it back |
| `pull`/`push`: "header.json is for workflow X, not Y" | The dir holds another workflow's dataset of the same name | `--workflow X`, or another `--dir` |
| `run`: "resume it with the same settings" | `--resume` with a different version or repeats | Re-run with the experiment's settings, or start a new experiment |
| `run`: needs `--access-token` / a wallet | Missing orchestrator or vault credential | Have the user set `$CEF_ACCESS_TOKEN` (ROC → Access) |
| `run`: "has no deployment set — deploy it once before evaluating a version" | A non-live version can only be pinned on a deployed workflow | `cef deploy` first (see `cef:deploy`) |
| A run ends `error` with "has more than 100 pages" | The run's context is too long to read in full | Report it; do not treat it as a timeout |
| `migrate`: "Not marked migrated: fix the skipped datasets" | A legacy dataset could not be copied | Report each skipped dataset and its reason |

Rule of thumb: a refusal protects someone's work. Explain which work, then let
the user pick the override.
