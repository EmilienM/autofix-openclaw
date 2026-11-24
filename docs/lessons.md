# Lessons from building an agent-platform SDLC loop

Platform behaviors discovered while deploying this on OpenClaw Enterprise /
OpenShell. Each one cost a debugging cycle; none were obvious from the docs.

## Triggering

- Dedicated-composition agents force `automations.scope: "caller"`,
  `allowCommandJobs: false`, and `cron.triggers.enabled: false`. The last one
  disables automation *triggers* that run model-written commands in the
  gateway process, but **timed automations that run ordinary turns still
  execute in the harness**. An in-agent schedule is therefore possible; an
  in-agent script job is not.
- The agent-side automations tool was broken in our runtime (its output
  failed the declared schema; gateway/runtime version drift). Automations
  are managed through the gateway CLI instead. The CLI wants job UUIDs, not
  job names; "automation not found" usually means you passed the name.
- Long HTTP-chat turns from outside the cluster die at roughly 30 seconds:
  the platform route has the default router timeout and kills the SSE stream
  while the model thinks silently. Streaming is required regardless; for
  anything long, trigger the turn inside the gateway.

## Credentials

- The native-harness exec tool runs commands in a sanitized environment that
  strips variable names matching token/secret patterns, so credentials
  injected by the platform never reach commands the agent runs. Files on the
  persistent workspace are the workaround.
- Credential files must not have trailing newlines; several consumers treat
  the newline as part of the secret and fail closed.
- A fine-grained GitHub PAT needs Contents read/write for branch pushes even
  when it already has Pull requests read/write; `git push` is a
  contents-write operation and GitHub refuses it with a generic
  "resource not accessible" error.

## Networking

- Sandbox egress required two cooperating changes: the endpoint must be in
  the installation-level delegation ceiling, and the agent needs a policy
  addition within that ceiling (saved, then applied to the running revision).
  The ceiling lives in an immutable installation secret, so adding an
  endpoint is a full installation change.
- `toolBinaries` (which binaries may carry credential sources to their
  endpoints) is separate from the ceiling. If your HTTP client is not in
  toolBinaries, credential sources cannot open the network for it.
- The runtime image had no `jq`; python3 was present but not network-fenced.
  Keeping curl as the only HTTP client preserves the per-binary fences.

## Runtime

- The persistent workspace survives redeploys; the runtime home does not.
  Anything that must survive (credentials, scripts, state, cloned repos)
  belongs on the workspace volume, and git identity/credential config should
  be repo-local (set after clone) rather than global.
- A wedged sandbox worker (exec returning UNAVAILABLE, environment stop
  failing in a loop) does not recover on its own, and deleting the pod does
  not get it recreated. Redeploying the agent (a new revision) re-provisions
  the sandbox; the workspace survives.
- One control-plane node hosting several agent gateways can run out of
  schedulable memory; agents compete for slots and stopping one agent is how
  another fits. Capacity is part of the design whether you plan for it or
  not.

## Process lessons

- Two independent reviews of the helper scripts caught a leftover dependency
  on a tool that does not exist in the runtime (every push would have
  failed) and a queue-wedging state machine (tickets that failed once would
  be retried forever, head-of-line blocking everything). Deterministic glue
  deserves the same review discipline as product code.
- Driving the model through a fixed runbook with scripts for every
  secret-touching step worked well: the tick behaved identically across
  runs, and the sandbox network logs were enough to verify the agent did
  what the runbook said (the Jira query appeared under the expected fence
  rule, nothing else was touched).

## Subagent sessions (known limitation)

Per-ticket subagent sessions need runtime support that our version does not
have yet. The gateway tool baseline exposes `sessions_spawn` and friends, but
on a dedicated worker the tool refuses the model's calls: the system prompt
describes the full gateway schema while the worker accepts only `task` and
`label`, and the rejection does not name the offending field. Restating the
worker schema in the prompt did not help on our runtime; the spawn call ends
without creating a child session, and the caller can hit a session lifecycle
race on top. The upstream fix (openclaw/openclaw#203292, "let agents on a
dedicated worker start child sessions") plus its announced follow-ups need to
land in the runtime image before the orchestrator/fixer split works.

Until then, one ticket per tick already gives a per-ticket session: each
tick run is an isolated session a human can open and continue. What is
missing is the split itself and ticket-key-named sessions.
