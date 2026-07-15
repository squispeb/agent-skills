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

- `agent-orchestration`: selects models, writes self-contained prompts, and coordinates parallel agents.
- `worktree-spawn`: creates worktrees, copies local configuration, and selects models for subagents.

## Ownership

Everything in this repository is authored and maintained by us. Skills that
were already installed globally but whose authorship could not be established
are intentionally excluded; `~/.agents/skills` is an installation directory,
not the source of truth. Review changes here with Git, then install a selected
revision through `bunx skills`.

The `manifest.json` file records the intended ownership and portable format of
each skill. It is deliberately separate from harness installation metadata,
which is not reliably preserved by global installs.
