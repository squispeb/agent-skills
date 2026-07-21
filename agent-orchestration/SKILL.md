---
name: agent-orchestration
description: >-
  Orchestrates subagents from any harness by assessing task difficulty,
  selecting the model whose intelligence and taste clear the task's floor,
  spawning via `opencode run`, and verifying output before accepting it.
  Use when delegating a subtask, choosing an agent model, deciding how hard
  a task is, or coordinating parallel agents.
---

# Agent Orchestration

Spawn subagents with `opencode run` -- a plain CLI command, so any harness with a shell tool (Claude Code, Codex, Cursor, Gemini CLI, opencode itself) orchestrates the same way. Workflow: assess the task's floors, pick the cheapest model that clears them, write a self-contained prompt, verify the output before accepting it. Do not delegate a task smaller than the prompt needed to describe it -- do it inline.

## Roles: Orchestrator and Worker

Spawned subagents load this same skill; without a role boundary a worker re-orchestrates and recurses indefinitely. Only the orchestrator -- the agent talking to the user -- may call `opencode run`.

- Every delegated prompt MUST begin with `ROLE: worker (<archetype>).`
- **If your own task prompt begins with `ROLE: worker`, you are the worker.** Execute directly with your own tools and return the requested output. Never spawn subagents, even if the task looks decomposable -- if it is too big to do directly, return a message saying it needs decomposition and let the orchestrator split it.

**Structural enforcement (opencode):** this skill's directory ships `worker.md`, a restricted agent definition that denies the `task` tool and all `opencode *` bash commands, so a worker cannot recurse even if it ignores its prompt. Install once per machine, then spawn with `--agent worker`:

```bash
cp <this-skill-dir>/worker.md ~/.config/opencode/agents/worker.md
```

Keep the `ROLE:` prefix regardless -- it is the only protection on other harnesses or when the agent file is missing.

## Assessing the Required Intelligence

Classify the task by observable traits before picking a model. The floor is the minimum intelligence rating the model must have.

| task traits | intelligence floor |
|-------------|--------------------|
| Locked spec, mechanically verifiable output: renames, migrations, boilerplate, format conversions | 4 |
| Standard implementation with existing patterns to copy and clear acceptance criteria | 8 |
| Ambiguous scope, cross-file reasoning, or judgment calls you cannot pre-specify in the prompt | 9 |
| Expensive-to-reverse decisions: architecture, data models, public APIs, security audits | 10 |

Anything user-facing (UI, copy, API design) additionally needs taste >= 7. Mixed tasks take the highest floor present; when axes conflict for work that ships, resolve intelligence, then taste, then cost.

## Models and Archetypes

Pick the cheapest row whose intelligence AND taste clear the task's floors. Rankings 1-10, higher = better; cost reflects actual paid cost, not list price.

| archetype (floor) | model | cost | int | taste | prompt must include |
|-------------------|-------|------|-----|-------|---------------------|
| executor (4) | `opencode-go/kimi-k2.7-code` | 10 | 4 | 4 | locked spec verbatim, exact file paths, output format |
| implementer (8) | `openai/gpt-5.6-luna` | 9 | 8 | 6 | acceptance criteria, test expectations, patch format |
| ui / copy (taste >= 7) | `anthropic/claude-sonnet-5` | 5 | 5 | 7 | design constraints, tokens/components to match, accessibility requirements |
| reviewer (9) | `openai/gpt-5.6-terra` | 7 | 9 | 8 | artifact to review inline, review dimensions, findings-list format |
| architect (10) | `openai/gpt-5.6-sol` | 6 | 10 | 9 | full context and constraints; output decision + rationale + rejected alternatives |

Alternates when a default does not fit: `anthropic/claude-fable-5` (int 9, taste 9, cost 2 -- most expensive per intelligence unit), `anthropic/claude-opus-4-8` (int 7, taste 8, cost 4 -- review and planning), `openai/gpt-5.5` (int 8, taste 5, cost 8 -- bulk work, less efficient than luna).

Escalate without asking when a cheaper model will not meet the bar -- escalating costs less than shipping mediocre work.

## Verifying and Escalating

Never accept subagent output unread; check it against the acceptance criteria from the prompt. Diagnose failures before retrying:

1. **Spec failure** -- competent output, wrong problem. Re-prompt the SAME model with the correction; in opencode, continue the worker's session so it keeps context: `opencode run --session <id> "Correction: ..."` (IDs via `opencode session list`).
2. **Capability failure** -- clear spec, wrong or shallow output. Escalate one intelligence tier (kimi -> luna -> terra -> sol) and re-run the same prompt.
3. **Repeated capability failure** -- two escalations without acceptable output means the task is not delegable as specified. Do it yourself or decompose it into smaller tasks with lower floors.

## `opencode run`

```bash
opencode run "ROLE: worker (implementer). Implement the validation rules from the attached spec in src/foo.ts and add focused tests in src/foo.test.ts. Acceptance: tests pass, public API unchanged. Return a patch." \
  -f docs/validation-spec.md -m openai/gpt-5.6-luna --title "validation-rules"
```

Every prompt must be self-contained: role prefix, exact file paths, task boundaries, explicit acceptance criteria (you verify against them), and the output format (file content, patch, JSON, findings list). Never reference "the previous conversation" or other context the worker cannot see.

Flags for constructing workers:

| flag | use |
|------|-----|
| `-m provider/model` | model per the table above |
| `--agent worker` | spawn through the restricted worker agent (see Roles) |
| `--dir <path>` | run in another directory or git worktree without `cd` |
| `--file <path>` (`-f`) | attach spec/reference files instead of pasting them |
| `--format json` | machine-readable event stream for parsing output |
| `--title "<label>"` | keeps parallel sessions attributable in `opencode session list` |
| `--auto` | auto-approve anything not denied -- required unattended; pair with `--agent worker` so denies hold |
| `--session <id>` (`-s`), `--fork` | continue/fork a worker session for re-prompting |
| `--variant <effort>` | provider-specific reasoning effort |

## Parallel Work

Only parallelize independent tasks: no overlapping file edits, no dependency on each other's output. Redirect each worker to its own log or the interleaved stdout is unattributable.

```bash
opencode run "ROLE: worker (ui). task A ..." -m anthropic/claude-sonnet-5        > /tmp/task-a.log 2>&1 &
opencode run "ROLE: worker (executor). task B ..." -m opencode-go/kimi-k2.7-code > /tmp/task-b.log 2>&1 &
wait
# verify each log separately before accepting
```
