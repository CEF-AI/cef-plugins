# Connector steps (`action` nodes)

A workflow's `action` step does one thing in an external system (send a Slack
message, read a HeyReach campaign) through a vault connection. The runner
publishes `connector.action.requested` to `connection:<id>` and waits for the
connector's answer the way it waits on an agent.

```ts
{ id: "post", kind: "action", params: {
    connection: "cn_01a1…",                         // which connection
    action: "send_message",                         // what to do
    input: { channel: "={{ $json.channel }}", text: "hi" },
} }
```

## Taking the connection from the run's data

**Requires `@cef-ai/agent-sdk` with templated connections (the release after 5.15.0).**

When the connection depends on the run, for example a campaign's HeyReach
account stored in its `campaign_setup` row, write `connection` as a mapping and
list every connection the step may use in `connections`:

```ts
{ id: "campaign-status", kind: "action", params: {
    connection: "={{ $json.heyreachConnectionId }}",
    connections: ["cn_heyreach_main", "cn_heyreach_second"],
    action: "get_campaign",
    input: { campaignId: "={{ $json.heyreachCampaignId }}" },
} }
```

- `connections` is required with a mapping, and holds literal ids only.
  `validate` refuses `a templated connection needs connections: [the ids it may resolve to]`
  or `connections must list literal connection ids`.
- The runner resolves the mapping against the step's input and trims it. It
  sends nothing, and fails the step (and so the run), when the id:
  - resolves empty: `<node>: connection resolved empty from <template>`;
  - is not listed: `<node>: connection <id> from the run's data is not one this step may use (<list>)`.
- A plain `connection` (no leading `=`) works exactly as before; `connections`
  is ignored there.

## Access

The vault only lets the workflow use connections it was granted. ROC writes
those grants from the graph when you Deploy or Run: one per literal
`connection`, and one per id in a templated step's `connections`. Every listed
id must be a connection in the vault the workflow runs in.

The `cef` CLI does not write connector access. After a code deploy that adds an
id to `connections`, press **Run** on the workflow's page in ROC to update its
access. Until then the vault refuses the send, and the step fails.

Deploy a workflow with a templated connection with `cef deploy`, not ROC's
Deploy button: ROC's Deploy links the workflow to ROC's bundled runner, and
until that runner is agent-sdk 5.16.0 or later ROC refuses the step with
"deploy this workflow with `cef deploy`".
