# claude-skills

Personal [Claude Code](https://claude.com/claude-code) configuration: custom skills, subagents,
and global settings. This repo is checked out directly as `~/.claude`.

## Layout

| Path | Contents |
| --- | --- |
| `skills/` | Custom skills, one directory per skill with a `SKILL.md` |
| `agents/` | Subagent definitions for the `Agent` tool (see [`agents/README.md`](agents/README.md)) |
| `repos/` | Repo profiles: the per-codebase facts the generic skills and agents load (see [`repos/README.md`](repos/README.md)) |
| `settings.json` | Global Claude Code settings (model, hooks, theme, plugins) |

Skills and agents are **repo-agnostic**. Each one first matches the current repo to a
profile in `repos/` (by remote URL, then marker files) and reads only the topic files it
needs. The repo's own `CLAUDE.md` / `AGENTS.md` / ADRs still win. Profiles exist for:

| Profile | Repo |
| --- | --- |
| `repos/savi/` | Savi sled monorepo, GitLab `savicontrols/savi` (DSP and main lines) |
| `repos/iot/` | FarmBot `iot` monorepo, GitHub `farmbot-monitoring-solutions/iot` |

To support a new codebase, add a profile; don't fork a skill.

### Skills

| Skill | Purpose |
| --- | --- |
| `work-ticket` | Implement a ticket end to end: readiness, base branch, tracer bullet, tests from AC, gates, specialist reviewers, Codex pass, hand-off to closeout. Runs inline so it can delegate |
| `push` | Commit, push, and open an MR/PR on GitLab or GitHub with the repo's remotes, base branch, naming and template; link it in Jira |
| `adversarial-review` | Review a change against its real base and the ticket's AC, with the specialist reviewers and an iterative Codex adversarial loop |
| `review-follow-up` | Address review feedback on an MR/PR: verify and triage each thread, fix the class, re-validate, push, draft replies |
| `ticket-closeout` | Verify AC against the change, post the closeout comment, tick boxes, carry findings into downstream tickets |
| `ticket-write` | Draft, rewrite, or critique tickets against the shared standard (`skills/ticket-write/ticket-standard.md`) |
| `backlog-sync` | Publish approved drafts to Jira, add real blocking links, and drift-check a design doc against its epic |
| `epic-loop` | One iteration of the per-epic loop (sync, triage, closeout, restack, next ticket); state in `~/.claude/state/epic-loop/<EPIC>/` |
| `qa-ticket` | Deploy the MR/PR or RC build to the test environment and QA a ticket with live browser testing, system checks, and regression testing |
| `qa-acceptance-criteria` | Derive missing, testable acceptance criteria from a ticket and its MR/PR |
| `release-regression` | Regression-test a release candidate by ticket or as a general regression; GO / NO-GO report |
| `release` | Cut the next release branch and bump versions, or walk the release finalization checklist |
| `mergeback` | Merge an older release forward as a mechanical merge plus one adaptations commit, then open the MR/PR |
| `standup` | Build a standup from actual Jira, GitLab and GitHub activity since the previous workday |

### Agents

`impact-tracer`, `conventions-reviewer`, `async-bug-hunter`, `spec-runner`, `spec-author`,
`ticket-critic`, `legacy-source-scout`, `pr-stack-planner`, `mergeback-auditor`,
`release-ticket-triage`. See [`agents/README.md`](agents/README.md) for when to use each and
how they compose.

## What is not tracked

`.gitignore` is an allowlist: everything under `~/.claude` is ignored except the paths above.
That keeps session transcripts (`projects/`), `history.jsonl`, logs, caches, plugin installs,
file history, plans, shell snapshots, skills synced from claude.ai (`skills/synced/`), and
per-epic loop state (`state/`) out of the repo. Machine-local files such as `settings.local.json` and `.env` are ignored too.

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
