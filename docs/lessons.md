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

## Subagent sessions

Per-ticket fixer sessions need runtime support for dedicated workers to
start child sessions; earlier runtime builds refused the model's
`sessions_spawn` calls (the system prompt described the full gateway schema
while the worker accepts only `task` and `label`, and the rejection did not
name the offending field). The upstream fix (openclaw/openclaw#203292) makes
plain spawns work. Spawned children run on the same worker: they get the
sandbox, its exec tool, and the network fences, so a fixer subagent can run
the same helper scripts as the orchestrator.

Three behaviors to know once spawning works:

- **Session labels are unique.** Spawning with a label that an earlier
  attempt's session still holds fails with "label already in use". The
  orchestrator runbook retries with a numeric suffix (`-2`, `-3`, ...), which
  keeps each attempt in its own continuable session instead of failing the
  ticket.
- **Children can stall on worker capacity.** A spawned child may sit queued
  until a worker slot frees up; the parent sees a slow or
  unknown-outcome spawn that later completes. Do not read an early
  non-response as a failure.
- **Detect results by label, not session key.** Spawned children get
  generated `dashboard:cloud-*` session keys; the label you passed is a
  separate field. Watching for the label in the session list (or the
  terminal status on the child) is the reliable check — grepping keys for
  the label finds nothing and makes working spawns look like failures.

## Worker capacity (the parallel-subagent limit)

Spawned subagents are not free: each one is a worker environment on the
agent's sandbox node, and the node refuses further sessions at its capacity.
The limit is `runtime.nativeOpenClawSessionCapacity` in the Installation
(default 8; our stage had been set to 2, so an orchestrator plus one fixer
exhausted it). Three things made this confusing to diagnose:

- The orchestrating agent's own turn also occupies a slot, so usable
  concurrency is capacity minus one.
- Idle workers are not retired automatically upstream, so completed
  sessions can hold slots for a while.
- A capacity-refused spawn does not queue; the parent's turn can die with a
  node-disconnect instead of failing that one ticket cleanly.

Changing the value is an Installation change, and a sandbox only picks it up
after a redeploy. Size it to peak subagent concurrency plus one, and keep a
retry-on-capacity-error rule in the orchestrator runbook as a safety net.
