---
name: worktree-spawn
description: >-
  Guides git worktree setup (which git-ignored files to copy) and how to spawn
  opencode subagents via `opencode run` with the right model for the task.
  Use when creating a git worktree, setting up a parallel workspace, or deciding
  which model to use when spawning an agent for a subtask.
---

# Worktree Setup & Agent Spawning

## Worktree: Files to Copy

When creating a new git worktree, these files are not carried over (git-ignored or project-local) and must be copied manually from the source worktree:

| File | Why |
|------|-----|
| `.env` | gitignored -- DB credentials, secrets |
| `.env.local` | gitignored -- local overrides, API keys |
| `.env.bak` | gitignored -- backup env (copy if exists) |
| `opencode.json` | project-level AI config (MCP keys, agents) |
| `portless.json` | gitignored -- local portless routing config (copy if exists) |

Full worktree creation flow:

```bash
BRANCH="feature/my-branch"
# Sanitize branch name for use as directory name and portless subdomain
WORKTREE_NAME="$(echo "$BRANCH" | sed 's|[/.]|-|g')"
ROOT="$(git rev-parse --show-toplevel)"
WORKTREE="$ROOT/../$WORKTREE_NAME"

git worktree add "$WORKTREE" -b "$BRANCH"

# Copy git-ignored and project-local config
cp "$ROOT/.env"          "$WORKTREE/.env"
cp "$ROOT/.env.local"    "$WORKTREE/.env.local"
[ -f "$ROOT/.env.bak" ] && cp "$ROOT/.env.bak" "$WORKTREE/.env.bak"
cp "$ROOT/opencode.json" "$WORKTREE/opencode.json"
[ -f "$ROOT/portless.json" ] && cp "$ROOT/portless.json" "$WORKTREE/portless.json"

# Initialize codegraph index if the source worktree has one
[ -d "$ROOT/.codegraph" ] && codegraph init "$WORKTREE"
```

### tunutri -- portless URL patching

tunutri uses **portless** (`portless.json` sets `name: "tunutri"`) to route worktrees:

- Main app: `https://tunutri.localhost:1355`
- Worktree `feature-xyz`: `https://feature-xyz.tunutri.localhost:1355`

Full tunutri worktree creation (run from the main worktree):

```bash
BRANCH="feature/my-branch"
WORKTREE_NAME="$(echo "$BRANCH" | sed 's|[/.]|-|g')"
ROOT="$(git rev-parse --show-toplevel)"
WORKTREE="$ROOT/../$WORKTREE_NAME"

git worktree add "$WORKTREE" -b "$BRANCH"

cp "$ROOT/.env"          "$WORKTREE/.env"
cp "$ROOT/.env.local"    "$WORKTREE/.env.local"
[ -f "$ROOT/.env.bak" ] && cp "$ROOT/.env.bak" "$WORKTREE/.env.bak"
cp "$ROOT/opencode.json" "$WORKTREE/opencode.json"
[ -f "$ROOT/portless.json" ] && cp "$ROOT/portless.json" "$WORKTREE/portless.json"

# Derive Portless hostname prefix from the worktree's current branch
PORTLESS_PREFIX="$(git -C "$WORKTREE" branch --show-current | sed -E -e 's|^t3code/||' -e 's|[/.]|-|g' | tr '[:upper:]' '[:lower:]')"
PORTLESS_URL="https://${PORTLESS_PREFIX}.tunutri.localhost:1355"

# Patch NEXT_PUBLIC_BASE_URL, BETTER_AUTH_URL, UPLOADTHING_CALLBACK_URL
sed -i \
  -e "s|^NEXT_PUBLIC_BASE_URL=.*|NEXT_PUBLIC_BASE_URL=${PORTLESS_URL}|" \
  -e "s|^BETTER_AUTH_URL=.*|BETTER_AUTH_URL=${PORTLESS_URL}|" \
  -e "s|^UPLOADTHING_CALLBACK_URL=.*|UPLOADTHING_CALLBACK_URL=${PORTLESS_URL}/api/uploadthing|" \
  "$WORKTREE/.env.local"

echo "Ready: cd $WORKTREE && bun dev -> $PORTLESS_URL"
```

`UPLOADTHING_CALLBACK_URL` is also overridden at runtime via `PORTLESS_URL` injected by portless (see `src/app/api/uploadthing/route.ts`), so uploads work even if the `sed` step is skipped.

---

## Picking the Right Model

Rankings 1-10: higher = better. Cost reflects what I actually pay (OpenAI has really generous limits), not list price. Intelligence is how hard a problem you can hand the model unsupervised. Taste covers UI/UX, code quality, API design, and copy.

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

Rules:

- Bulk/mechanical (clear-spec impl, data analysis, migrations): `kimi-k2.7-code`; escalate to `openai/gpt-5.6-luna` if output misses the bar -- luna is the sweet spot for coding tasks.
- Anything user-facing (UI, copy, API design) needs taste >= 7: `anthropic/claude-sonnet-5` or higher.
- Review / second opinion / complex reasoning: `openai/gpt-5.6-terra` or `anthropic/claude-fable-5`.
- Hardest unsupervised problems, architecture: `openai/gpt-5.6-sol`.
- Cost is a tie-breaker only; when axes conflict for anything that ships, intelligence > taste > cost.
- These are defaults -- override without asking if cheaper output does not meet the bar. Escalating costs less than shipping mediocre work.

---

## Spawning Agents via `opencode run`

```bash
# General syntax
opencode run "self-contained task prompt" -m <model-id>

# Bulk/mechanical (effectively free)
opencode run "implement X per spec in src/foo.ts -- return only the final file" \
  -m opencode-go/kimi-k2.7-code

# User-facing work
opencode run "redesign the BookingForm component, match existing Tailwind tokens" \
  -m anthropic/claude-sonnet-5

# Efficient coding task
opencode run "implement X per spec in src/foo.ts -- return only the final file" \
  -m openai/gpt-5.6-luna

# High-intelligence unsupervised task
opencode run "audit the auth flow in src/auth/ for security issues, output findings as a list" \
  -m openai/gpt-5.6-sol

# Review / second opinion / complex reasoning
opencode run "review this implementation plan for edge cases: ..." \
  -m openai/gpt-5.6-terra
```

### Prompt requirements for `opencode run`

Prompts must be **self-contained** -- the spawned agent has no prior context:

- Include exact file paths and relevant constraints.
- Specify the expected output format (file content, diff, JSON, list).
- Do not reference "the previous conversation" or "the task above".

### Subagent delegation inside an opencode session

Use `@execution` or `@ui` to delegate within a session (model fixed by `opencode.json`):

- `@execution` -- `opencode-go/kimi-k2.7-code` -- fast, cheap, code-focused
- `@ui` -- frontend/visual work, uses session default

For a task that needs a specific model not in the agent config, use `opencode run` directly instead.

### Parallel agents

Run multiple agents in parallel by backgrounding each call:

```bash
opencode run "task A ..." -m anthropic/claude-sonnet-5 &
opencode run "task B ..." -m opencode-go/kimi-k2.7-code &
wait
```
