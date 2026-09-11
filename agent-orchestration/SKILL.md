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

| tier | route | rationale |
|---|---|---|
| trivial/easy | `executor` via `task` → `openai/gpt-5.6-luna` | Plus is $0 marginal, and luna is fastest at 109.4 t/s |
| medium | primary: `executor` via `task` → `openai/gpt-5.6-luna`; CLI worker via `opencode run --agent worker -m <model>` → `opencode-go/glm-5.3-flash`, `opencode-go/deepseek-v4-flash`, or `opencode-go/deepseek-v4.1-flash` | Use luna first; use the three Go lanes for routine implementation and bounded investigation |
| hard | CLI `worker` via `opencode run --agent worker -m <model>` → `opencode-go/glm-5.3` or `openai/gpt-5.6-sol` | Use the stronger models when the task needs more judgment; sol is Zen-only and its promotion expires Sep 18, 2026 |
| critical | primary `architect` via `task` → `anthropic/claude-opus-5`; Fable via CLI `worker -m anthropic/claude-fable-5-1`; Opus fallback | Use the architect lane first; Fable is credit-metered on Pro |

The floors are keyed to cluster membership, not artificial two-point boundaries: the cheap
cluster is 46.9–48.4 agentic, while the top cluster is 57.8–59.2. `task` accepts no model
argument, so hard-tier routing must use `opencode run --agent worker -m`; never spawn a
worker via `task`, because it inherits the orchestrator model. Escalate without asking when
a cheaper worker will not meet the bar.

| `subagent_type` | model | agentic | intel | $/task | min/task | budget |
|---|---|---|---|---|---|---|
| `executor` | `openai/gpt-5.6-luna` | 46.9 | 38 | $0.20/$1.20 per 1M | — | Go, $15/mo + Plus $0 marginal; fastest at 109.4 t/s |
| `architect` | `anthropic/claude-opus-5` | 59.2 | 51 | $0 | — | Pro subscription; updated 2026-09-10; saves the $15 Go bucket; `glm-5.3` demoted to fallback |
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
| `opencode-go/glm-5.3-flash` | 58.2 | 42 | $0.15/$0.50 per 1M | — | the old executor row, retained as the Go credit-saving lane; its bucket is now $60/mo ≈ 500 tasks |
| `opencode-go/deepseek-v4-flash` | 48.4 | 35 | $0.15/$0.60 per 1M off-peak | — | a second cheap bucket ($30/mo) once the flash allowance is spent |
| `opencode-go/deepseek-v4.1-flash` | TBD — no AA composite Agentic score published; vendor-reported DeepSWE 74.2 and AutomationBench 54.8 are secondary | **40 (AA Intelligence Index v4.3, canonical);** vendor-reported Terminal-Bench 2.1 90.6 is secondary; 190.1 tok/s | AA peak $0.30/$1.20 per 1M; Go-only off-peak $0.15/$0.60 per 1M; $15/mo bucket; no Zen entry | — | medium overflow after `glm-5.3-flash` is spent, preserving the glm bucket |
| `openai/gpt-5.6-sol` | 57.8 | 47 | Zen $2/$10 through Sep 18, 2026, then $4/$15; not on Go | — | the hard-tier Zen lane while the promotion lasts |
| `anthropic/claude-opus-5` | 59.2 | 51 | Zen $5/$25 metered | — | the ceiling on both axes, for critical calls worth paying for |
| `opencode-go/glm-5.2` | 43.1 | 39 est. | $1.40/$4.40 per 1M | — | architect fallback with a $60 allowance |

## Budget

OpenCode Go ($10/mo) meters in dollars, not requests: **$12 per 5 hours, $30 per week,
$60 per month** — and each model carries its own monthly allowance inside that total,
which is the constraint that actually bites. Go's published request counts assume ~150
output tokens per request; real worker tasks emit 10k-55k, so ignore them.

`glm-5.3` and `glm-5.3-flash` are Go-only — Zen tops out at GLM 5.2 — and each draws its
own $15, so `executor` and `architect` never compete for the same bucket; together they
cap at $30 of the $60 ceiling. The flash bucket is $60/mo, or about 500 tasks at $0.12,
instead of the old ~125-task estimate. Stretch the buckets by keeping `architect` advisory
and capping `steps`. When a bucket runs dry there is no paid fallback for that model, so
move `executor` to the ChatGPT Plus route, which is free, and `architect` to `glm-5.2`,
which carries a $60 allowance. `deepseek-v4.1-flash` has AA Intelligence Index 40 (v4.3),
190.1 tok/s, and AA peak pricing of $0.30/$1.20 per 1M; it is Go-only with $0.15/$0.60
off-peak pricing, no Zen entry, and its own $15 monthly bucket. Its AA composite Agentic
score remains TBD; vendor benchmarks are secondary. Sol's Zen promotion is $2/$10 through Sep 18, 2026, then
reprices to $4/$15, roughly 2x.

Figures retrieved 2026-09-10 using the v4.3 scale break from the
[Agentic Index](https://artificialanalysis.ai/models/capabilities/agentic), the
[Intelligence Index](https://artificialanalysis.ai/#intelligence-tabs),
[Go](https://opencode.ai/docs/go/), and [Zen](https://opencode.ai/docs/zen/).
`deepseek-v4.1-flash` AA facts were updated 2026-09-10 from its
[model page](https://artificialanalysis.ai/models/deepseek-v4-1-flash): Intelligence Index 40 (v4.3),
190.1 tok/s, and $0.30/$1.20 per 1M peak pricing; its composite Agentic score remains TBD.
`glm-5.3-flash` has a confirmed independent Agentic Index entry of 58.2 from Aug 27; its
v4.3 Intelligence Index score is 42. Prices come from the Go and Zen docs; per-task costs
are omitted where the fresh figures provide token rates rather than a comparable task
estimate.

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
