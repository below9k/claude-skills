# Repo profiles

The skills and agents in `~/.claude` are generic. The facts that differ from one codebase to
another live here, as one directory per repo: the tracker, the forge, branch and commit
rules, the test commands, the architecture rules, the test environment, and the release
process.

```
repos/
  README.md            ← this file: how to load a profile, and the section contract
  savi/README.md       ← Savi monorepo (GitLab savicontrols/savi): profile + index
  savi/*.md            ← topic files, read only when a skill or agent needs that topic
  iot/README.md        ← FarmBot iot monorepo (GitHub farmbot-monitoring-solutions/iot)
  iot/*.md
```

## Loading the profile (every skill and agent does this first)

1. From the repo root, run `git remote -v` and `ls`.
2. Pick the profile whose `match.remotes` substring appears in a remote URL. If no remote
   matches, pick the one whose `match.markers` all exist at the root.
3. Read that profile's `README.md`, then only the topic files your task needs (the index at
   the top of each profile says which file holds what).
4. **The repo's own instructions win.** If `CLAUDE.md`, `AGENTS.md`, `docs/adr/` or a
   repo-local `.claude/` or `.agents/` skill states a rule, it beats the profile. Profiles
   hold what the repo doesn't write down (or doesn't write down on every branch).
5. **No profile matches:** work it out from the repo (its docs, `package.json`/`Makefile`
   scripts, CI config, `git log` conventions, remote host) and say in your output that no
   profile was found. If you learned something durable, offer to start a profile.

Name the profile (and, where a profile has variants such as Savi's DSP and main lines, the
variant) in your output.

Facts marked *observed <date>* are snapshots, not guarantees. Re-check them before relying
on them, and update the profile when they turn out wrong.

## Section contract

A profile's `README.md` has front matter and these sections. Leave a section out when it
doesn't apply. Skills look for them by heading.

```markdown
---
name: <short name>
match:
  remotes: ["<host/path substring>", ...]
  markers: ["<path at repo root>", ...]
---

# <Repo>

## Topic files          ← index: file → what it holds → who reads it
## Tracker              ← Jira site, project keys, AC field, ticket key regex, status names
## Forge and remotes    ← GitLab or GitHub; fork workflow or origin; remote names; CLI/MCP
## Branches             ← base-branch rules, branch name format, exceptions
## Commits and PRs      ← message style, PR/MR template, links, labels
## Validation           ← build / lint / test commands, narrowest first; known traps
## Agents               ← which generic agents are worth running here, and when
```

Topic files are free-form, but use these names when they fit, so skills can find them:

| File | Holds | Read by |
| --- | --- | --- |
| `architecture.md` | house rules a reviewer checks; where blast radius hides | `conventions-reviewer`, `impact-tracer`, `async-bug-hunter`, `work-ticket` |
| `testing.md` | runner paths, spec style, how to tell a real pass from a vacuous one | `spec-runner`, `spec-author`, `work-ticket` |
| `runtime.md` | running the app locally and proving a change in it | `work-ticket`, `review-follow-up` |
| `qa-env.md` | test server, artifacts, deploy steps, existing fakes | `qa-ticket`, `release-regression` |
| `release.md` | release branches, versions, tags, mergebacks, finalization | `release`, `mergeback`, `mergeback-auditor`, `release-ticket-triage`, `release-regression` |
| `tickets.md` | ticket-writing specifics beyond the shared standard | `ticket-write`, `backlog-sync`, `ticket-critic`, `ticket-closeout`, `epic-loop` |

## Adding a repo

Copy the section contract into `repos/<name>/README.md`, fill in what you know, and leave
the rest out. A thin profile is fine; skills fall back to inferring from the repo. Add
facts as they come up: a correction from the user about "how we do it here" belongs in the
profile, not in a skill.
