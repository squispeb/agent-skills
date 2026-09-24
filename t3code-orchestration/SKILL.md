---
name: t3code-orchestration
description: >-
  Orchestrates OpenCode subagents inside t3code so every spawn stays attached
  to the thread: routes work to task-tool lanes, handles usage-limit
  exhaustion with an ordered fallback, and runs `opencode run` only as a
  supervised, visible-to-the-user degraded mode. Use when running in t3code
  (the t3-code MCP tools are available) and delegating, choosing a worker
  model, or recovering from a subagent usage-limit error.
---

# t3code Orchestration

t3code renders a subagent only when it is a native `task` tool part or a child session
whose `parentID` points into the thread. Anything else is invisible: no Agents panel row,
no token usage, no error surfacing, and no permission forwarding. The transport decides
visibility, and this skill exists to keep every spawn visible.

It reuses the `executor`, `operator`, `architect`, and `worker` agent definitions shipped by
`agent-orchestration` (installed in `~/.config/opencode/agents/`) and defines no agents
of its own. The prompt contract — the `ROLE: worker (<archetype>).` prefix,
self-contained prompts, and the `BLOCKED` / `ACCEPTANCE` sections — and the secrets
handling are the same as in `agent-orchestration`.

## Setup

Every spawn needs two things settled before it runs: which model it uses and which tools
it sees.

Pin the built-in subagents. OpenCode's rule is that "subagents will use the model of the
primary agent that invoked the subagent", so an unpinned `explore` spawned from an Opus
orchestrator runs on Opus and spends the expensive budget on grep work.

Keep the external MCP servers on one lane. Each enabled server injects its full tool
schema into every model call whether or not the task uses it. Measured on a "reply OK"
probe, above a 12.5k baseline: vercel adds 139k input tokens, resend 72k, railway 21k,
trigger 11k, and all four together 268k. A one-line code fix cost 748k this way, and
that is what drains a lane's usage limit. A `deny` removes a server's tools from the
model's list entirely; an `allow` makes them callable (both verified per server).

So `executor`, `architect`, and `worker` deny all four, and the `operator` lane allows
all four for tasks that genuinely need a service. The built-ins need the denies too, in
`~/.config/opencode/opencode.json`:

```json
"agent": {
  "explore": {
    "model": "opencode-go/deepseek-v4.1-flash",
    "permission": { "vercel_*": "deny", "railway_*": "deny", "resend_*": "deny", "trigger_*": "deny" }
  },
  "general": {
    "model": "openai/gpt-5.6-luna",
    "permission": { "vercel_*": "deny", "railway_*": "deny", "resend_*": "deny", "trigger_*": "deny" }
  }
}
```

Keep the deny list in step with the `mcp` block: a newly added server reaches every
worker until you deny it here and in the agent files, and should be allowed in
`operator`. Small code-reading servers such as `codegraph` are worth leaving allowed.
Only configure built-ins your version ships
(`opencode agent list`): an entry for one that does not exist, such as `scout` before it
was added, silently creates a new promptless agent instead.

t3code spawns a fresh `opencode serve` per session, so agent and config changes apply to
new threads, not to a thread that is already running. Confirm with
`opencode debug agent explore` that the resolved `model` is the pin and the MCP prefixes
appear as `deny`.

## Transport

`task` is the only transport that keeps a spawn attached to the thread, so use it for
every delegation. Call it with:

- `subagent_type`: the lane from the table below.
- `description`: 3-5 words; it becomes the Agents panel title.
- `prompt`: the full self-contained worker prompt.
- `task_id`: optional; set it only to resume the same worker for a correction.

`task` takes no model argument, so the `subagent_type` is how you pick the model.

Independent work runs in parallel only when every `task` call is in the same assistant
message. Each `task` call blocks until its worker returns, so issuing one, waiting, and
then issuing the next runs them one after another even if you meant them to overlap. Never
parallelize overlapping edits, and never background anything.

When you report which model a spawn used, report the model the child actually ran on — the
one the Agents panel shows, or the `model` in the task result — never the one this skill's
table expects. The two differ exactly when setup is broken, which is when the report
matters.

## Lanes

| work | lane | transport | model |
|---|---|---|---|
| exploration, search, reading | `explore` | `task` | pinned in Setup (`opencode-go/deepseek-v4.1-flash`) |
| implementation, bounded fixes, tests | `executor` | `task` | `openai/gpt-5.6-luna` |
| work that needs Vercel, Railway, Resend, or Trigger.dev tools: status, logs, deploys, email, runs | `operator` | `task` | `openai/gpt-5.6-luna` |
| expensive-to-reverse decisions: architecture, data model, public API, security | `architect` | `task` | `anthropic/claude-opus-5` |

Ask `explore` for `file:line` leads rather than summaries, so you read only those lines
yourself instead of paying your own model to search. `architect` is advisory only, with
no write tools and no shell; implement its decision with `executor`.

Route to `operator` only when the task needs one of those services, since every call it
makes carries about 268k input tokens of schemas. Split mixed work: `operator` fetches
the logs or status, `executor` changes the code. `operator` treats reads as routine but
returns BLOCKED for any deploy, send, delete, or config change the prompt does not name
explicitly with its target, so name the exact action when you want one.

## Usage-Limit Exhaustion

A subagent error that mentions a usage limit — `The usage limit has been reached`,
`Go usage limit exceeded` — or carries HTTP 429 means that model is exhausted for the
rest of the session, even when the error says it is retryable.

- Do not retry the lane, and do not resume the failed `task_id` on the same model.
- Never run probe loops ("Reply with exactly: OK" across models) to find remaining quota.
  Each probe spends quota and time, and a probe against a dead lane can hang.
- Keep a running note of exhausted models and tell the user which lane died.
- For implementation work, fall back in this order: `executor` via `task` →
  `worker` via `opencode run -m opencode-go/glm-5.3-flash` → `worker` via
  `opencode run -m opencode-go/deepseek-v4.1-flash` → stop and ask the user.
- Architect work never falls back to a weaker model silently: stop and ask the user.

Moving to an `opencode run` lane enters the degraded mode below. Say so to the user in
one sentence before the first such spawn, because those workers will not appear in the
thread.

## Degraded Mode: `opencode run`

A bash-spawned `opencode run` starts its own server and creates a root session with no
`parentID`, so t3code cannot see it and you are its only supervisor. That changes what
you have to do:

1. **Foreground only.** Never `nohup`, never `&`, never detach. A background worker has
   no supervisor and no timeout, and its result arrives after you have moved on.
2. **Bound it with `timeout`,** and set the bash tool's own timeout above it. A provider
   error does not reliably end the process: one measured worker hit
   `Go usage limit exceeded` and then hung for the full 3600 s shell timeout.
3. **Use `--format json` and check the events for an error** before trusting the output.
   `opencode run` can exit 0, or not exit at all, when the provider errored.
4. **Always pass `--agent worker --dir <repo root> -m <model> --title <short-slug>`,**
   or `--agent operator` for service work.
   Without `--agent worker` the run falls back to the default agent and loses every
   denial. Write long prompts to a file under `/tmp/opencode/` and pass `"$(cat <file>)"`.
5. **A usage-limit error here applies the exhaustion rules:** mark the model dead and move
   down the fallback order.
6. **Verify the ACCEPTANCE section yourself** by running the checks the worker claims,
   since t3code shows none of its work.

```bash
timeout 1800 opencode run "$(cat /tmp/opencode/s3a-prompt.md)" \
  --agent worker --dir "$REPO" -m opencode-go/glm-5.3-flash \
  --title s3a-service-routes --format json > /tmp/opencode/s3a.jsonl 2>&1
echo EXIT=$?
```

To resume a degraded worker for a correction, use `opencode run -s <session-id> ...` with
the same hygiene.

## Recursion

Only the user-facing orchestrator delegates. A prompt beginning with `ROLE: worker`
executes directly and never calls `task` or runs `opencode`; the shipped agent
definitions deny both.
