# Agent Skills

Portable skills for AI coding agents and harnesses.

This repository is the source of truth for the skills authored by us. Each
skill follows the standard directory layout with a `SKILL.md` file, so the
repository can be installed by `bunx skills` and distributed to supported
harnesses including Antigravity, Codex, Cursor, Gemini CLI, OpenCode, and Zed.

## Install

Install every skill globally:

```bash
bunx skills add squispeb/agent-skills -g --all
```

Install selected skills:

```bash
bunx skills add squispeb/agent-skills -g \
  --skill agent-orchestration,worktree-spawn \
  --agent '*'
```

Use `--copy` when an environment cannot use symlinks. The default installer
behavior is preferred because the harness installation stays linked to the
source package during local development.

## Skills

- `agent-orchestration`: assesses task difficulty, selects models, writes self-contained worker prompts, and coordinates parallel agents.
- `worktree-spawn`: creates worktrees, copies local configuration, and selects models for subagents.

## Agent definitions

`agents/opencode/worker.md` is a restricted opencode agent used by
`agent-orchestration` for structural loop prevention: it denies the `task`
tool and any `opencode *` bash command, so a spawned worker cannot recurse
into spawning its own subagents. `bunx skills` does not install agent
definitions, so copy it once per machine:

```bash
mkdir -p ~/.config/opencode/agents
cp agents/opencode/worker.md ~/.config/opencode/agents/worker.md
```

Then spawn workers with `opencode run "..." --agent worker -m <model>`.

## Ownership

Everything in this repository is authored and maintained by us. Skills that
were already installed globally but whose authorship could not be established
are intentionally excluded; `~/.agents/skills` is an installation directory,
not the source of truth. Review changes here with Git, then install a selected
revision through `bunx skills`.

The `manifest.json` file records the intended ownership and portable format of
each skill. It is deliberately separate from harness installation metadata,
which is not reliably preserved by global installs.
