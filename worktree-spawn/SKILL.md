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

## Spawn Inside the Worktree

Use the `agent-orchestration` skill to select the model, construct the worker prompt, and verify its output. This skill adds one worktree-specific invariant: every repository worker must start inside the worktree it will modify.

```bash
opencode run "ROLE: worker (implementer). Work only in this repository. ..." \
  --agent worker \
  --dir "$WORKTREE" \
  -m openai/gpt-5.6-luna
```

`--dir "$WORKTREE"` roots the worker's sandbox in the target checkout. Do not start it in the main checkout and give it absolute paths into another worktree; direct cross-worktree access is blocked and can force a fragile export/apply handoff through `/tmp`.

For parallel worktrees, launch one worker per worktree and give every worker its own `--dir`. Tasks that need results from another worktree should return them to the orchestrator, which verifies and combines them without granting workers cross-worktree access.
