---
name: agent-orchestration
description: >-
  Orchestrates subagents from any harness: classifies the task, spawns the
  cheapest worker that clears it via OpenCode's native `task` tool, tracks the
  metered model budget, and verifies both the spawn and the output.
  Use when delegating a subtask, choosing an agent model, deciding how hard
  a task is, or coordinating parallel agents.
---

# Agent Orchestration

Classify the task, spawn the cheapest worker that clears it, verify the worker actually
ran as configured, then verify its output against the acceptance criteria. Do not
delegate a task smaller than the prompt needed to describe it.

Three workers exist. Resist adding more: every extra archetype is another definition
that can drift out of sync with this file.

## Install

This skill ships the agent definitions in `agents/`. It is the source of truth; the
copies under `~/.config/opencode/agents/` are generated artifacts and are overwritten.

```bash
cp ~/.agents/skills/agent-orchestration/agents/*.md ~/.config/opencode/agents/
```

Agent files load at startup. A running TUI keeps the old definitions until it is
restarted; each `opencode run` subprocess picks them up immediately.

## Delegation Transport

In OpenCode, use the native `task` tool directly — it preserves child-session
lifecycle events for the host. If `task` is available, never wrap delegation in Bash
and never invoke `opencode run`.

Call `task` with:

- `subagent_type`: `executor`, `architect`, or `worker`. OpenCode resolves this exact
  name to the global agent definition.
- `description`: a short 3-5 word label for the child-session title.
- `prompt`: the full self-contained worker prompt, beginning with `ROLE: worker (<archetype>).`
- `task_id`: optional child session ID from a prior result; set it only to continue that
  same worker session for a correction.

Pass nothing else. Foreground execution is the default, and `task` accepts no model or
effort argument: both come from the agent definition, so picking the `subagent_type` is
how you pick the model. To run a task on a different model, edit the definition or add
one — never assume an argument overrode it.

For independent tasks, issue multiple `task` calls in the same assistant turn to run
them in parallel. Never parallelize overlapping edits or dependent tasks, and never
substitute background Bash processes for parallel `task` calls.

## Roles and Recursion

Only the user-facing orchestrator may delegate.

- Every delegated prompt MUST begin with `ROLE: worker (<archetype>).`
- If your own prompt begins with `ROLE: worker`, execute directly and NEVER call `task`,
  spawn a subagent, or run `opencode`. If the work is too large, report that it needs
  decomposition.
- `executor` and `architect` are `mode: subagent` (task-spawned only). `worker` is
  `mode: all` because `opencode run --agent` rejects subagent-mode agents — it silently
  falls back to the default agent and drops every tool denial. Never change it.
- All three deny `task`. `executor` and `worker` also deny `opencode *` and `git push`,
  written in every spelling including plugin-rewritten forms (`rtk git push *` as well as
  `git push *`), because patterns match the command string after `tool.execute.before`
  hooks run. Prefer a structural deny over a pattern list where you can: `architect`
  denies `bash` outright, since `edit: deny` removes the write tools but still leaves
  `echo > file` reachable through the shell. Treat a pattern deny as a backstop rather
  than proof, and verify it fires.
- Keep the `ROLE:` prefix: it is the only safeguard on other harnesses or when agent
  configuration is missing.

## Choosing a Worker

| `subagent_type` | model | agentic | intel | $/task | min/task | budget |
|---|---|---|---|---|---|---|
| `executor` | `opencode-go/glm-5.3-flash` | 58.2 | 57.5 | ~0.12 est | 12.5 | Go, $15/mo ≈ 125 tasks |
| `architect` | `opencode-go/glm-5.3` | 59.1 | 59.5 | 0.68 | 7.7 | Go, $15/mo ≈ 22 tasks |
| `worker` | none — caller pins with `-m` | — | — | — | — | whatever you pin |

Use `executor` for anything with a locked spec and mechanically checkable output, and
for ordinary implementation with clear acceptance criteria. Use `architect` only for
expensive-to-reverse decisions — architecture, data models, public APIs, security — and
never to write code; it has no write tools and no shell, and its output tokens dominate
its cost.
Anything user-facing still needs your own review: no model here is picked for taste.

`architect` is not automatically an escalation — compare its scores against whatever model
you are running on. When you are the stronger model, delegate to it for what it is
cheaper at: long, token-heavy reading you would otherwise spend your own context on. When
you are weaker, delegate the decision itself.

Swap a model into a definition when a default does not fit — `task` cannot select one
dynamically, so edit the agent file rather than silently using the wrong model:

| alternate | agentic | intel | $/task | min/task | use when |
|---|---|---|---|---|---|
| `openai/gpt-5.6-luna` | 46.9 | 52.3 | 0.049 | 2.6 | the fast, free lane: $0 marginal on ChatGPT Plus and the quickest here, for latency-bound or high-volume batches. Costs 11 agentic points |
| `opencode-go/deepseek-v4-flash` | 48.4 | 51.8 | ~0.04 est | 7.1 | a second cheap bucket ($30/mo) once the flash allowance is spent |
| `openai/gpt-5.6-sol` | 57.8 | 60.9 | 0.953 metered, $0 on Plus | 3.8 | the smart free lane: highest intelligence here at no marginal cost on ChatGPT Plus, and fast with it |
| `anthropic/claude-opus-5` | 59.2 | 63.1 | 2.337 | 7.3 | the ceiling on both axes, for calls worth paying for |

Escalate without asking when a cheaper worker will not meet the bar.

## Budget

OpenCode Go ($10/mo) meters in dollars, not requests: **$12 per 5 hours, $30 per week,
$60 per month** — and each model carries its own monthly allowance inside that total,
which is the constraint that actually bites. Go's published request counts assume ~150
output tokens per request; real worker tasks emit 10k-55k, so ignore them.

`glm-5.3` and `glm-5.3-flash` are Go-only — Zen tops out at GLM 5.2 — and each draws its
own $15, so `executor` and `architect` never compete for the same bucket; together they
cap at $30 of the $60 ceiling. Stretch them by keeping `architect` advisory and capping
`steps`. When a bucket runs dry there is no paid fallback for that model, so move
`executor` to the ChatGPT Plus route, which is free, and `architect` to `glm-5.2`, which
carries a $60 allowance.

Figures retrieved 2026-08-28 from the
[Agentic Index](https://artificialanalysis.ai/models/capabilities/agentic), the
[Intelligence Index](https://artificialanalysis.ai/#intelligence-tabs),
[Go](https://opencode.ai/docs/go/), and [Zen](https://opencode.ai/docs/zen/).
`glm-5.3-flash` shipped 2026-08-26 and has no Agentic Index entry yet, so its 57.5 is an
Intelligence Index score — do not read it across the agentic column. Prices come from the
Go docs, which bill it at $0.15/$0.50 per 1M; models.dev advertises half that, and the
billing source wins. Per-task costs assume measured output tokens plus ~20k fresh input
and ~50k cached-read input; they are order-of-magnitude, not billing.

## Reasoning Effort

Set effort with `reasoningEffort` in the agent definition, never with `variant` — a
variant is silently discarded unless the agent's own pinned model is the session model
and the name exists in that model's `variants` map, and `task` overrides it with the
parent's value regardless.

```yaml
model: openai/gpt-5.6-luna
reasoningEffort: high   # none | minimal | low | medium | high | xhigh | max
```

Confirm it landed under `options` with `opencode debug agent <name>` before relying on
it. The provider rejects an invalid value and names the set it accepts, so trust that
error over model metadata.

Leave effort unset unless you specifically want to trade judgment for latency: luna's
unpinned default scores 46.9 on the agentic index against 44.4 at xhigh and 41.0 at high.

## Prompt Contract

Every prompt is self-contained: the `ROLE:` prefix, exact paths, explicit boundaries,
explicit acceptance criteria, and the output format. Never reference conversation
context the worker cannot see.

The workers are instructed to stop and return `BLOCKED: <question>` rather than resolve
an ambiguity, and to return an `ACCEPTANCE:` section marking each criterion MET or
NOT MET with `file:line` or command-output evidence. Write criteria that can actually
carry evidence, or that section becomes theater.

## Secrets and Interactive Prompts

A permission set to `ask` is a deadlock inside a subagent: nothing can answer the prompt,
so the tool call blocks until something aborts it. opencode ships `read` on `*.env` as
`ask`, so a worker told to open `.env` hangs rather than failing — measured once at 26
minutes, surfaced to the parent only as `Task cancelled`.

So audit for `ask` before delegating (`opencode debug agent <name>`) and convert anything
a worker could reach into `deny`, which fails in milliseconds and lets the worker return
BLOCKED. All three definitions deny `*.env` for this reason.

Workers that need a secret load it through bash, which keeps the value out of the model's
context entirely:

```bash
set -a; . ./.env; set +a
psql "$DIRECT_URL" -c 'select 1'
```

Instruct workers never to echo or log a secret, and to report only what they can prove
about it — that it is set, its length, its host.

When `task` returns `Task cancelled`, read the child session before retrying; a retry
walks into the same wall and costs the same stall twice.

## Verify the Spawn, Then the Output

Verify the spawn first — a misconfigured agent silently downgrades to the default agent,
carrying its permissions instead of yours. Three commands, none of which spend a token:

- `opencode agent list` — every agent with its mode. A missing entry means the file did
  not parse. A `(subagent)` entry cannot be used with `--agent`.
- `opencode debug agent <name>` — resolved model, permissions, and `options`, so you can
  see whether a deny or a `reasoningEffort` actually landed.
- `opencode debug config` — the merged config, which also reveals agents defined inline
  in `opencode.json` rather than as files. Those are easy to forget and can be pinned to
  a model you thought you retired.

Then prove enforcement instead of assuming it: spawn a worker that attempts the denied
command and confirm it is blocked. Read the output rather than the exit status —
`opencode run` exits 0 even when the provider returned an error.

Then check the output against the prompt and diagnose before retrying:

1. **Spec failure** — call `task` again with the prior `task_id` and the same
   `subagent_type` plus a correction, so the worker keeps context. In fallback mode,
   `opencode run --session <id> "Correction: ..."`.
2. **Capability failure** — re-spawn on a stronger worker: `executor` → `architect` for
   the decision, then `executor` again to implement it.
3. **Repeated failure** — after two escalations, do it yourself or decompose it into
   smaller tasks.

A `BLOCKED` reply is not a failure; answer the question and re-spawn.

## CLI Fallback Only

For a non-OpenCode harness, or when `task` is genuinely unavailable, use the restricted
`worker` agent rooted in the target repository or worktree. Never spawn in one worktree
and instruct the worker to edit another.

```bash
opencode run "ROLE: worker (executor). Implement validation in src/foo.ts and add focused tests. Acceptance: tests pass, public API unchanged. Return a patch." \
  --agent worker --dir "$TASK_DIR" -m openai/gpt-5.6-luna --title "validation-rules"
```

`-m provider/model` selects the model dynamically here — the one place that is possible.
`--file/-f` attaches references, `--format json` emits machine-readable events, and
`--session/-s` resumes for corrections.
