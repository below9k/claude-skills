# claude-skills

Personal [Claude Code](https://claude.com/claude-code) configuration: custom skills, subagents,
and global settings. This repo is checked out directly as `~/.claude`.

## Layout

| Path | Contents |
| --- | --- |
| `skills/` | Custom skills, one directory per skill with a `SKILL.md` |
| `agents/` | Subagent definitions for the `Agent` tool (see [`agents/README.md`](agents/README.md)) |
| `settings.json` | Global Claude Code settings (model, hooks, theme, plugins) |

### Skills

| Skill | Purpose |
| --- | --- |
| `gitlab-code-review` | Review a branch/commit against its release base branch and Jira acceptance criteria, with an adversarial Codex pass |
| `gitlab-push` | Commit, push to the user's fork, and open GitLab MRs against the detected release branch |
| `work-ticket` | Implement a Jira ticket end to end |
| `qa-ticket` | QA a Jira ticket against its acceptance criteria and regression areas |
| `qa-acceptance-criteria` | Derive missing, testable acceptance criteria from a ticket and its MR/PR |

### Agents

`slice-planner`, `impact-tracer`, `savi-invariant-reviewer`, `async-bug-hunter`, `spec-runner`,
`spec-author`, `sled-e2e`, `qa-ticket`, `qa-acceptance-criteria`. See
[`agents/README.md`](agents/README.md) for when to use each and how they compose.

## What is not tracked

`.gitignore` is an allowlist: everything under `~/.claude` is ignored except the paths above.
That keeps session transcripts (`projects/`), `history.jsonl`, logs, caches, plugin installs,
file history, plans, shell snapshots, and skills synced from claude.ai (`skills/synced/`) out
of the repo. Machine-local files such as `settings.local.json` and `.env` are ignored too.

To track something new, add a `!/<path>` line to `.gitignore`.

## Setup on a new machine

```sh
# Fresh machine (no ~/.claude yet)
git clone https://github.com/below9k/claude-skills ~/.claude

# Existing ~/.claude
cd ~/.claude
git init -b main
git remote add origin https://github.com/below9k/claude-skills
git fetch origin
git reset origin/main          # keeps local files; review with git status/diff
```

Hook commands in `settings.json` reference `~/.config/iterm2/cc-status`; adjust or remove them
on machines without that script.

## License

GPL-3.0 — see [LICENSE](LICENSE).
