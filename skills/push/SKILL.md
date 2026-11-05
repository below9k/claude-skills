---
name: push
description: >
  Use for essentially every commit, push, branch publication, or merge
  request / pull request operation, on GitLab or GitHub. Applies whenever the
  user asks to commit, push, create or update a branch, or open an MR/PR.
  Loads the repo profile for the forge, remotes (fork workflow or origin),
  base-branch rules, branch naming, commit style, and PR template; falls back
  to the repo's own conventions. Never assumes main is the base, never pushes
  to protected/release branches by default, never force-pushes without
  authorization, and links the MR/PR on the Jira ticket.
---

# Commit, push, and open an MR/PR

Request:

$ARGUMENTS

The default sequence:

1. Load the repo profile and the repo's own conventions.
2. Determine the base branch.
3. Create or reuse a correctly named branch.
4. Review the change.
5. Commit in the repo's style.
6. Push to the right remote.
7. Open the MR/PR against the base.
8. Link it on the ticket and report.

## 1. Conventions

Follow `~/.claude/repos/README.md`: match the repo to a profile and read its **Forge and
remotes**, **Branches**, and **Commits and PRs** sections. Then read what the repo says about
itself: `CONTRIBUTING.md`, `CLAUDE.md`/`AGENTS.md`, `.github/pull_request_template.md` or
`.gitlab/merge_request_templates/`, and repo-local commit or PR skills (e.g. a
`conventional-commit` or `raise-pr` skill). **The repo wins over the profile.**

With no profile, infer:
- **Forge** from the remote host (`git remote -v`): GitLab → GitLab MCP / `glab`; GitHub →
  `gh`.
- **Workflow**: two remotes where one is the user's (`origin` → fork, `upstream`/`base` →
  canonical) means a fork workflow: push to the fork and open the MR/PR into canonical. One
  remote means push to `origin`.
- **Branch names and commit style** from recent history (`git log --oneline -30`,
  `git branch -r`).

## 2. Base branch

Don't assume `main`, `master`, or `develop`. Use the profile's rule (e.g. "fork from the
release branch the work belongs to", or "stack on the blocker's branch"). Otherwise inspect:

```sh
git branch -a; git remote -v; git log --oneline --decorate -20
```

Use the current branch's upstream, existing MRs/PRs for the ticket, and the ticket itself as
evidence. If several bases are plausible, ask. Never pick one arbitrarily.

Update the base before branching. With uncommitted changes, branch straight from the
updated remote base so the changes stay in the tree:

```sh
git fetch <canonical>
git switch -c <feature-branch> <canonical>/<base-branch>
```

If `git switch` refuses because the changes conflict, stop and tell the user rather than
stashing or discarding their work.

## 3. Branch

Name it per the profile or the repo's convention. Take ticket keys from the request, the
current branch, commits, or the ticket. **Never invent ticket numbers**; ask. Keep the
description short and kebab-case.

If the user is already on a correctly named branch, keep using it. If the branch exists
remotely, continue from it rather than creating a duplicate. Never overwrite someone
else's branch.

## 4. Review the change

```sh
git status; git diff; git diff --cached
```

Commit only the changes that belong to the request; leave unrelated changes in the tree.
Check for secrets, credentials, private generated files and stray debug output. Run the
profile's quick gates (or the repo's lint/test) when practical, and don't claim you did if
you didn't.

## 5. Commit

- Use the repo's style (the profile's **Commits and PRs**: e.g. ticket-prefixed
  `PRJ-567 Fix audio subscription cleanup`, or conventional `feat(pkg): …` with no key).
- Commit under the user's configured identity (`git config user.name` / `user.email`). Don't
  change it. Some CIs trigger only for the user's own identity; the profile says when that
  is functional, not cosmetic.
- Branch under review: add commits on top. Don't amend, squash or rebase unless asked.

## 6. Push

- Fork workflow: `git push -u <fork-remote> <branch>`. Origin workflow:
  `git push -u origin <branch>`. If the profile gives a special push command (a credential
  helper, an HTTPS URL), use it.
- Never push to the canonical repo's protected or release branches unless the user
  explicitly asks.
- Diverged remote: don't force-push. Inspect the divergence and propose the safe step.
  `--force` needs explicit authorization; prefer a pinned
  `--force-with-lease=<branch>:<expected sha>`.

## 7. Open the MR/PR

- Source: the pushed branch (on the fork, in a fork workflow). Target: the canonical repo's
  base branch from step 2. Verify both before creating.
- Use the repo's template or PR skill when there is one; otherwise: what changed, why,
  notable implementation details, testing performed (only what was actually run), and known
  limitations or follow-ups. Add whatever the profile asks for on top (e.g. "Stacked on #n",
  a Refs line, the adversarial-review buckets).
- Title per the profile (often the ticket key plus summary).
- Create it as the user. If the authenticated CLI or MCP account isn't the user, don't
  silently create it under another identity.
- No CLI or MCP available: give the web URL for creating it. Never claim it was created
  when it wasn't.

## 8. Link and report

Check whether the ticket already links the MR/PR (remote links, dev panel, a comment). If
not, add it as a remote link, or a short comment. Don't transition the ticket.

```
Profile:  <name | none>
Base:     <base branch>
Branch:   <feature branch>
Remote:   <remote pushed to>
Commit:   <hash> <subject>
MR/PR:    <URL>
Jira:     <linked | already linked | not linked, because …>
```

If a step didn't complete, say which and why.

## When to ask

Ordinary requested operations (branch, commit, push to the user's fork or feature branch,
open the MR/PR) need no extra confirmation. Ask when the ticket key or base branch is
missing or ambiguous; the destination remote is unclear; unrelated changes can't be
separated safely; the push would rewrite remote history; or the operation would touch
someone else's work.
