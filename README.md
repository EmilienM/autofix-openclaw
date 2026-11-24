# autofix-openclaw

An AI SDLC prototype: an autonomous agent that watches a Jira queue, picks up
labeled tickets, implements the fix, and opens a draft pull request for human
review. It runs as a first-class agent on an OpenClaw Enterprise (OCE) cluster
instead of inside a CI pipeline, which is the interesting part: the same
platform that hosts chat agents also hosts the software-development loop.

Proven end to end: a test ticket filed in Jira was claimed, fixed, committed
with sign-off, pushed, and opened as a draft PR, and the ticket got the PR
link back in a comment.

## How it works

```
every 2h (in-agent /automations job)
  └─ tick runbook (one turn in an isolated session)
       1. jira-search        → tickets labeled "autofix-openclaw", oldest first
       2. claim              → add "autofix-openclaw-wip" label (double-pickup guard)
       3. jira-get           → read the ticket (treated as untrusted input)
       4. allowlist check    → only the one approved repo, fail closed
       5. clone + branch     → autofix/<TICKET>, clean worktree, repo-local git config
       6. implement + test   → the LLM writes the fix, runs cheap tests if present
       7. commit             → signed off (DCO), subject prefixed with the ticket key
       8. push-and-pr        → push + draft PR + state record as one unit
       9. report             → comment on the ticket with the PR URL
```

## Design principles

**The model never touches secrets.** Credentials live as files on the agent's
persistent workspace, and every secret-touching operation (Jira search, label,
comment, git push, PR creation) is a hardened shell script. The agent calls the
scripts; it never reads the credential files, and the scripts never print what
they read. Deterministic plumbing is code, LLM judgment is confined to
interpreting the ticket and writing the fix.

**Enforcement is layered soft to hard.** Runbook guardrails (advisory) sit on
top of sandbox network fences (per binary and host: only curl may reach Jira
and the GitHub API, only git's HTTP helper may reach github.com), which sit on
top of a fail-closed allowlist check in the push script, which sits under a
fine-grained PAT scoped to one repository. The strongest control is innermost.

**Terminal states are recorded.** Every outcome (processed, skipped, failed,
stuck) is written to a state file, so a ticket that cannot be fixed leaves the
queue instead of blocking it. The Jira label state machine provides the
cross-run guard.

**One ticket per tick, drafts only.** Humans merge. The schedule also stays
off until someone decides to turn it on; that was a deliberate choice, not an
oversight.

## Repository layout

- `docs/install.md` — how to deploy this on your own cluster, and how to
  validate it before turning the schedule on
- `docs/design.md` — the design document (goals, non-goals, constraints,
  approach trade-offs)
- `docs/runbook.md` — the exact procedure the agent follows each tick
- `docs/lessons.md` — what building this taught us about running agents on
  OCE/OpenClaw (the gotchas are the valuable part)
- `scripts/` — the helper scripts the agent calls, plus `config.example.json`
  showing the configuration shape (JQL, repo allowlist, commit identity)

## Adapting it

The scripts assume a Jira Cloud instance and GitHub, with credentials staged
as `jira.env` (JIRA_URL, JIRA_EMAIL, JIRA_API_TOKEN) and `gh-pat` (a
fine-grained PAT with Contents and Pull requests read/write on the target
repo). The runbook assumes an OpenClaw agent with a persistent workspace at
`/sandbox/enterprise` and sandbox network fences for the three endpoints.
`mark-state` needs python3 for JSON; the scripts use curl for all HTTP so the
per-binary network fences hold.

## Status and caveats

The write path is proven (one real ticket to one real draft PR). Known gaps
before anything like production: the repo's test suite runs in the same
sandbox as the credential files (the PAT scope, not the fences, is what
actually bounds the blast radius), the automation is managed through the
gateway CLI because the agent-side automations tool has a schema drift bug in
this runtime, PRs are authored by a personal account rather than a bot
identity, and there is no iterate-on-review-feedback loop yet.
