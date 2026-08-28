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

`agent-orchestration/agents/` holds the opencode agent definitions the skill
delegates to. The skill directory is the source of truth; the copies under
`~/.config/opencode/agents/` are generated artifacts, so install with an
overwrite once per machine and after every skill update:

```bash
mkdir -p ~/.config/opencode/agents
cp ~/.agents/skills/agent-orchestration/agents/*.md ~/.config/opencode/agents/
```

Three definitions ship:

- `executor.md` — bounded implementation against a locked spec. Returns
  `BLOCKED` instead of resolving an ambiguity itself.
- `architect.md` — advisory only for expensive-to-reverse decisions. No write
  tools and no shell, so it cannot route around `edit: deny`.
- `worker.md` — the `opencode run` fallback for non-opencode harnesses. It is
  `mode: all` because `--agent` rejects subagent-mode agents and silently falls
  back to the default agent, dropping every denial.

All three deny the `task` tool so a spawned worker cannot recurse, deny
`opencode *` bash commands, and deny reading `*.env` — opencode ships that read
as `ask`, and an interactive prompt inside a subagent has nobody to answer it,
so it hangs instead of failing. Workers source secrets through the shell
instead. Agent files are read at startup: restart opencode after installing.

## Ownership

Everything in this repository is authored and maintained by us. Skills that
were already installed globally but whose authorship could not be established
are intentionally excluded; `~/.agents/skills` is an installation directory,
not the source of truth. Review changes here with Git, then install a selected
revision through `bunx skills`.

The `manifest.json` file records the intended ownership and portable format of
each skill. It is deliberately separate from harness installation metadata,
which is not reliably preserved by global installs.
