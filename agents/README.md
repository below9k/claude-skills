# Subagents

Agent definitions for the Claude Code `Agent` tool. Each file is one agent; the frontmatter
`description` is what the model matches against when deciding to delegate, so it says *when
to use it*, not just what it is.

Invoke by name, e.g. "use impact-tracer on the device-settings interface".

All agents are repo-agnostic. Each one starts by loading the matching **repo profile** from
`~/.claude/repos/` (see [`../repos/README.md`](../repos/README.md)), which supplies the
codebase's rules, runner paths, branch conventions and release process. The repo's own
`CLAUDE.md` / `AGENTS.md` / ADRs win over the profile. With no profile, an agent infers from
the repo and says so.

## Code agents

| Agent | Use it when |
| --- | --- |
| `impact-tracer` | Before editing a shared interface, key, service, or schema. Maps the blast radius; ranks silent runtime breakage first. Read-only. |
| `conventions-reviewer` | On any non-trivial diff. Checks the repo's documented house rules (profile `architecture.md`, `CLAUDE.md`, ADRs) and nothing else. |
| `async-bug-hunter` | On transport, worker, queue, driver, or connection-handling changes. Promise lifetime, timer leaks, reconnect, retry, partial failure. |
| `spec-runner` | After edits, before a commit. Runs the affected tests with a runner that actually works and triages every failure. Never improvises a runner. |
| `spec-author` | When a change lands untested, or a bug needs a regression test. Matches the neighbouring tests' framework and proves red-before-fix. |

`conventions-reviewer` answers "does this obey our rules"; `async-bug-hunter` answers "is
this correct". They're deliberately split, run in parallel, and stay short.

## Ticket and PR agents

| Agent | Use it when |
| --- | --- |
| `ticket-critic` | Before publishing a draft, before starting a ticket, or to audit an epic. Checks against `skills/ticket-write/ticket-standard.md`; returns blockers, fixes, and questions. Read-only. |
| `legacy-source-scout` | Rewrite work. Verifies numbers and behaviours against the legacy repo with `file:line`, and lists cases the doc omits. Read-only. |
| `pr-stack-planner` | Repos that stack PRs: choosing a base while blockers are in review, or after a PR in a stack merges. Prints restack commands; never runs them. |

## Release agents

| Agent | Use it when |
| --- | --- |
| `mergeback-auditor` | Before pushing a mergeback, or reviewing someone else's. Two-commit discipline, dropped source changes, migration ordering. Read-only. |
| `release-ticket-triage` | While finalizing a release. Proposes merge/retarget, fixVersion cleanup, and release-notes fixes. Read-only. |

## How they compose with the skills

```
ticket-write ── ticket-critic + legacy-source-scout ──▶ backlog-sync ──▶ Jira
     ▼
work-ticket ── ticket-critic (readiness) + pr-stack-planner (base)
     │         impact-tracer (risky boundary) → spec-author → spec-runner
     │         conventions-reviewer + async-bug-hunter (parallel) → Codex pass
     ▼
push ──▶ MR/PR ──▶ ticket-closeout (verify AC, comment, carry forward)
     ▼
review-follow-up (review waves)        epic-loop drives all of the above per epic

release (cut / finalize) ── mergeback + mergeback-auditor, release-ticket-triage,
                            release-regression
```

The QA skills (`qa-ticket`, `qa-acceptance-criteria`, `release-regression`) run in a forked
`general-purpose` context rather than through a dedicated agent, so they can't start
subagents; they do their own criteria derivation and code-test runs inline.
