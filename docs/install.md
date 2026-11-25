# Installing autofix-openclaw on your own cluster

These steps assume an OpenClaw Enterprise (OCE) installation with the
Kubernetes compute driver, operator access to the `occ` CLI, and `oc`/`kubectl`
access to the cluster. Adjust names, namespaces and endpoints to your
installation. Everything below was proven on a working deployment; the
platform-specific parts are called out where your installation may differ.

## 1. Deploy the agent

Create an OCC configuration for a dedicated agent with the native OpenClaw
harness and your model provider (an internal OpenAI-compatible endpoint needs
a harness override for the model provider protocol and base URL, and an
`openai`-type credential source with `authorized_base_url` matching it). Bind
the agent's service principal to the credential source with `operate`, grant
the gateway service account use of the SCC your installation requires, then
deploy. Verify the revision is active and the sandbox logs show the model
probe ready.

Refer to your platform's agent deployment documentation for the exact
payloads; the shape is: configuration with `agents.defaults` model catalog and
`agentRuntime.id: "openclaw"`, then agent create with
`executionMode: "dedicated"` and `harnessAuth` pointing at the model
credential source.

## 2. Open the sandbox network fences

The loop needs exactly three egress grants, each fenced to one binary:

- `curl` to your Jira instance (port 443)
- `curl` to `api.github.com`
- git's HTTP helper (`/usr/libexec/git-core/git-remote-http`) to `github.com`

How this works depends on your installation. On OCE, each grant must exist in
the installation-level delegation ceiling (`sandboxNetworkPolicyCeiling` in
the Installation document) before the agent can add it to its own sandbox
policy. Agent policy edits may need enabling first
(`occ agent policy editing <agent> --enabled`); then save the additions
(`occ agent policy update <agent> --file additions.json`, the file carries
`expectedVersion` and the full additions array) and apply them to the running
revision (`occ agent policy apply <agent> --revision <rev> --expected-version
N`). Verify from inside a turn that each endpoint answers.

## 3. Stage the workspace

The persistent workspace in the sandbox (commonly `/sandbox/enterprise`; it
survives redeploys, unlike the runtime home) holds everything:

```
.autofix/
  bin/            jira-search, jira-label, jira-comment, jira-get,
                  push-and-pr, git-credential-gh, git-setup, mark-state
                  (from scripts/ in this repository, chmod 755)
  creds/jira.env  JIRA_URL="https://your-instance.example"
                  JIRA_EMAIL="you@example.com"
                  JIRA_API_TOKEN=<token>     (chmod 600, no trailing newline)
  creds/gh-pat    <fine-grained PAT: Contents + Pull requests read/write
                  on the target repo>        (chmod 600, no trailing newline)
  config.json     from scripts/config.example.json, filled in
  state.json      {}
  skills/autofix-openclaw/SKILL.md   from docs/runbook.md (orchestrator)
  skills/autofix-openclaw/FIXER.md   from docs/fixer.md (spawned fixer)
```

Write the files into the sandbox with `oc cp` plus a chmod pass, or whatever
file-injection mechanism your platform documents. The JQL in `config.json`
defines eligibility: a label for the queue (`autofix-openclaw`), a label for
the in-progress claim (`autofix-openclaw-wip`, excluded by the JQL), and the
allowlisted repository URL.

## 4. Create the automation

If your runtime's agent-side automations tool works, create the schedule by
asking the agent. In runtimes where it does not (ours had a schema-drift
bug), use the gateway CLI from inside the gateway pod:

```
node /app/openclaw.mjs cron add --every 2h --session isolated --no-deliver \
  autofix-tick "Run the autofix-openclaw tick now. Follow the runbook at
  /sandbox/enterprise/.autofix/skills/autofix-openclaw/SKILL.md exactly,
  including its guardrails. Process the eligible tickets, at most four per
  tick."
```

Keep it disabled until validated (`openclaw cron disable <job-id>`; the CLI
wants job UUIDs, not names).

## 5. Validate before enabling

1. Dry run on an empty queue: `openclaw cron run <job-id>`, then
   `openclaw cron runs <job-id>` — expect a clean stop after the Jira search
   returns nothing, and no network activity beyond the search.
2. File a real test ticket against your target repository (trivial,
   well-described fix), label it, and run a tick. Expect: the claim label
   added, a branch pushed, a draft PR opened with the ticket key in the
   title, a comment on the ticket with the PR URL, and the ticket recorded
   as processed in `state.json`.
3. Review the PR, then decide whether to enable the schedule
   (`openclaw cron enable <job-id>`).

## Operating notes

- Trigger a tick manually: `openclaw cron run <job-id>`; history:
  `openclaw cron runs <job-id>`.
- A human can continue a completed tick by opening that run's session in the
  agent UI; the workspace (including the cloned repo) persists.
- Re-running a ticket needs three resets, because eligibility is guarded in
  two places besides Jira: remove the `autofix-openclaw-wip` label on the
  ticket (the JQL excludes it), run `reset-state KEY` to drop the local
  terminal-outcome record, and delete the remote `autofix/KEY` branch if a
  previous attempt pushed one (deleting it closes the PR opened from it;
  the retry's push is not forced). Then run a tick.
- If the agent is ever recreated, re-do the IAM binding, the SCC grant, the
  harness-override deploy, the network policy, and the workspace staging.
  Revision redeploys lose nothing but the runtime home.
- Long HTTP-chat turns from outside the cluster may hit your router's
  streaming timeout; trigger long turns in-gateway instead.
