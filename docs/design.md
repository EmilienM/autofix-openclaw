# autofix-openclaw design

## Intent

Prove that a platform-managed OpenClaw agent can run a piece of the AI SDLC
autonomously: monitor a Jira queue, pick up a ticket, fix the code, and send
a PR for human review. A simplified version of a CI-pipeline-based autofix
workflow, relocated onto the agent platform.

## Goals

- A scheduled agent loop: search Jira for tickets with a specific label, take
  the oldest unprocessed one, implement the fix, push a branch, open a draft
  PR, and comment on the ticket with the PR link.
- Label state machine: `autofix-openclaw` marks eligibility, adding
  `autofix-openclaw-wip` claims the ticket, and terminal outcomes are recorded
  in a state file so tickets leave the queue.
- Reuse the platform's model setup (native OpenClaw harness, an internal
  OpenAI-compatible model gateway).
- One real ticket driven end to end as the acceptance test.

## Non-goals

- No iterate-on-review-feedback loop, no triage bot, no CVE flow.
- One repository only, enforced by allowlist and PAT scope.
- No merging: PRs are drafts, humans merge.
- No bot identity (personal account for the prototype).

## Anti-goals

- The agent must never be able to push or open PRs anywhere except the
  allowlisted repository.
- No credentials through environment variables (the native-harness exec path
  strips token-named variables anyway, so this is forced).
- Do not run the CI orchestrator inside the sandbox; the agent itself is the
  coding agent.

## Constraints that shaped the design

- The agent runs the native harness on a dedicated runtime, which forces
  `automations.scope: "caller"` and disables script-payload automation
  triggers; timed automations that run ordinary turns still work, so the
  trigger is an in-agent automation rather than an external scheduler.
- Sandbox egress is fenced per binary and host. The loop needs exactly three
  grants: curl to the Jira API, curl to the GitHub API, and git's HTTP helper
  to github.com. Anything broader was refused.
- No `jq` in the runtime image, and the interpreter that is present is not
  network-fenced, so scripts use curl for all HTTP and python3 only for JSON.
- Credentials reach the sandbox as files on the persistent workspace volume,
  because environment-variable injection is stripped by the exec path. The
  files are readable by the agent itself; accepted prototype risk, mitigated
  by the allowlist, draft PRs, one ticket per tick, and a fixed JQL.
- The push script performs allowlist check, push, PR creation, and the state
  record as one unit, so a half-completed claim cannot silently wedge.

## Approaches considered

1. Run the existing CI-based autofix tooling inside the sandbox. Rejected: it
   is CI-shaped (child pipelines, spawns its own agent runtime, expects CI
   variables). Wrong runtime for an agent-platform agent.
2. A codex-harness agent, where credential environment variables propagate.
   Rejected for this deployment: the internal model gateway required an
   image patch that had not landed.
3. A native-harness agent with a simplified runbook and credentials as
   workspace files. Chosen: the only path with a proven model setup, and the
   file-based credential workaround is contained and explicit.

## Left to the implementer

Exact platform CLI calls, session keys, the runbook wording, and the failure
comment templates. See `runbook.md` for what the agent actually follows and
`lessons.md` for the platform behaviors that were discovered the hard way.
