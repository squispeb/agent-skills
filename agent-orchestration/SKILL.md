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

Spawn subagents with `opencode run`. It is a plain CLI command, so any harness with a shell tool (Claude Code, Codex, Cursor, Gemini CLI, opencode itself) can orchestrate the same way. The workflow is always: assess the task's required intelligence and taste, pick the cheapest model that clears both floors, write a self-contained prompt, then verify the output before accepting it.

Do not delegate a task smaller than the prompt needed to describe it — do it inline instead.

## Assessing the Required Intelligence

Classify the task by observable traits before touching the model table. The floor is the minimum intelligence rating the chosen model must have.

| task traits | intelligence floor |
|-------------|--------------------|
| Locked spec, mechanically verifiable output: renames, migrations, boilerplate from a template, format conversions | 4 |
| Standard implementation with existing patterns to copy and clear acceptance criteria | 8 |
| Ambiguous scope, cross-file reasoning, or judgment calls you cannot pre-specify in the prompt | 9 |
| Expensive-to-reverse decisions: architecture, data models, public APIs, security audits | 10 |

Two supplementary rules:

- **Taste floor:** anything user-facing (UI, copy, API design) additionally needs taste >= 7, regardless of the intelligence floor.
- **Mixed tasks:** take the highest floor among the traits present. When axes conflict for work that ships, resolve in this order: intelligence, then taste, then cost.

## Model Rankings

Rankings 1-10: higher = better. Cost reflects actual paid cost, not list price. Intelligence is how hard a problem can be handed to the model unsupervised. Taste covers UI/UX, code quality, API design, and copy.

| model | cost | intelligence | taste | best for |
|-------|------|-------------|-------|----------|
| `openai/gpt-5.6-sol` | 6 | 10 | 9 | hardest unsupervised problems, architecture decisions, complex reasoning |
| `anthropic/claude-fable-5` | 2 | 9 | 9 | hard unsupervised problems, architecture -- most expensive per intelligence unit |
| `openai/gpt-5.6-terra` | 7 | 9 | 8 | review, planning, multi-step reasoning, complex coding tasks |
| `anthropic/claude-opus-4-8` | 4 | 7 | 8 | review, planning, complex multi-step reasoning |
| `anthropic/claude-sonnet-5` | 5 | 5 | 7 | UI, copy, API design -- taste-sensitive work |
| `openai/gpt-5.6-luna` | 9 | 8 | 6 | coding tasks -- best token efficiency, impl, boilerplate, data transforms |
| `openai/gpt-5.5` | 8 | 8 | 5 | bulk/mechanical -- less efficient than luna for same capability tier |
| `opencode-go/kimi-k2.7-code` | 10 | 4 | 4 | high-volume file edits with a clear, locked spec |

## Selection Rules

Pick the cheapest model whose intelligence AND taste both clear the task's floors. The common cases resolve to:

- Floor 4 (bulk/mechanical, locked spec): `opencode-go/kimi-k2.7-code`.
- Floor 8 (standard implementation, boilerplate, data transforms): `openai/gpt-5.6-luna`.
- Taste >= 7 (user-facing UI, copy, API design): any model with taste 7 or above -- `anthropic/claude-sonnet-5` is the cheapest that qualifies; use a higher-intelligence model from that set if the task is also hard.
- Floor 9 (review, second opinions, planning, ambiguous scope): `openai/gpt-5.6-terra` or `anthropic/claude-fable-5`.
- Floor 10 (architecture, expensive-to-reverse decisions): `openai/gpt-5.6-sol`.

Treat these as defaults. Escalate without asking when a cheaper model will not meet the required quality bar -- escalating costs less than shipping mediocre work.

## Verifying and Escalating

Never accept subagent output unread. Check it against the acceptance criteria you put in the prompt. When it misses the bar, diagnose which failure it is before retrying, because the fixes are different:

1. **Spec failure** -- the output is competent but solves the wrong problem or misses a constraint. The prompt was ambiguous. Re-run the SAME model with a corrected, tighter prompt; escalating the model will not fix a bad spec.
2. **Capability failure** -- the spec was clear but the output is wrong, shallow, or incomplete. Escalate one intelligence tier (kimi -> luna -> terra -> sol) and re-run with the same prompt.
3. **Repeated capability failure** -- two escalations without acceptable output means the task is not delegable as specified. Do it yourself or decompose it into smaller tasks with lower floors.

## `opencode run`

```bash
opencode run "self-contained task prompt" -m <model-id>
```

Write every delegated prompt so the agent can act without prior conversation context:

- Include exact file paths, task boundaries, acceptance criteria, and relevant constraints.
- Specify the desired output format: file content, patch/diff, JSON, findings list, or another exact form.
- State the acceptance criteria explicitly -- you will verify against them when the agent returns.
- Do not reference "the previous conversation", "the task above", or other unavailable context.

Examples:

```bash
# Floor 4: mechanical edit, locked spec
opencode run "Implement the supplied spec in src/foo.ts. Preserve the public API. Return only the final file content." \
  -m opencode-go/kimi-k2.7-code

# Taste >= 7: user-facing UI work
opencode run "Redesign src/components/BookingForm.tsx. Match existing Tailwind tokens and preserve mobile accessibility. Return a patch." \
  -m anthropic/claude-sonnet-5

# Floor 8: standard implementation
opencode run "Implement the supplied validation rules in src/foo.ts and add focused tests in src/foo.test.ts. Return a patch." \
  -m openai/gpt-5.6-luna

# Floor 10: security-sensitive audit
opencode run "Audit src/auth/ for security vulnerabilities. Inspect all relevant files and return a severity-ranked findings list with file and line references." \
  -m openai/gpt-5.6-sol

# Floor 9: review / second opinion
opencode run "Review this implementation plan for correctness, edge cases, and missing tests: <plan>. Return a concise findings list." \
  -m openai/gpt-5.6-terra
```

## Subagent Archetypes

Do not depend on harness-configured agents (opencode `@execution`/`@ui`, Claude Code subagent types) -- they require per-project or per-user config that may not exist where you are running. Instead, define the subagent's role inside the prompt itself and spawn it with `opencode run`, which behaves identically from every harness.

Each archetype is a model choice plus what the prompt must contain for the output to be verifiable:

| archetype | model | prompt must include |
|-----------|-------|---------------------|
| executor (floor 4) | `opencode-go/kimi-k2.7-code` | the locked spec verbatim, exact file paths, output format |
| implementer (floor 8) | `openai/gpt-5.6-luna` | acceptance criteria, test expectations, patch format |
| ui / copy (taste >= 7) | `anthropic/claude-sonnet-5` | design constraints, existing tokens/components to match, accessibility requirements |
| reviewer (floor 9) | `openai/gpt-5.6-terra` | the artifact to review inline, the review dimensions, findings-list format |
| architect (floor 10) | `openai/gpt-5.6-sol` | full problem context and constraints, required output: decision plus rationale plus rejected alternatives |

## Parallel Work

Only run tasks in parallel when they are independent: no overlapping file edits and no dependency on each other's output. Redirect each agent's output to its own file, otherwise the interleaved stdout makes results unattributable.

```bash
opencode run "task A ..." -m anthropic/claude-sonnet-5      > /tmp/agent-task-a.log 2>&1 &
opencode run "task B ..." -m opencode-go/kimi-k2.7-code     > /tmp/agent-task-b.log 2>&1 &
wait

# Verify each result separately before accepting it
cat /tmp/agent-task-a.log
cat /tmp/agent-task-b.log
```
