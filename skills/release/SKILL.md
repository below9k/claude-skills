---
name: release
description: >
  Release-branch workflow, in two modes. "cut": create the next release
  branch (point, hotfix, rc, alpha, beta) from a release tip and bump the
  version strings. "finalize": walk the release finalization checklist —
  merge older active releases forward, cut the next branch, sweep open
  MRs/PRs (merge or retarget), clean up Jira fixVersions, prep release notes,
  verify the pipeline, wait for the test gate, merge back, tag, delete the
  release branch, and mark the Jira release Released. Repo specifics (version
  files, tag pattern, checklist details, people) come from the repo profile.
  Use when the user says "cut", "create" or "start" a release branch, "bump
  the version", or "finalize", "close out", "wrap up" or "ship" a release.
  Every outward-facing step needs explicit confirmation.
---

# Release

Request:

$ARGUMENTS

Load the repo profile (`~/.claude/repos/README.md`) and read its `release.md`. It holds
the product lines and branch naming, which file and variables carry the version, the
version-string forms per release kind, the commit message, where release branches live,
the shipped-tag pattern, the finalization checklist with its step numbers and specifics,
the gates, and who owns which decision. **If there is no `release.md`, stop and ask** how
this repo releases; don't invent a process.

Glossary: the **Release Version** is the one being finalized; the **Next Version** receives
the unfinished work. Pick the mode from the request; ask only if it's genuinely unclear.

## Confirmation rules

Read-only steps (inspection, triage, drafting) need no confirmation. **Everything marked ⚠
is outward-facing or destructive.** Present exactly what will happen (the list of MRs or
tickets, the before → after, the command) and wait for an explicit yes **for that step**.
Approval for one step doesn't carry to the next. Batch where it helps: one yes per table,
not per row. If a tool returns 403 or anything unexpected, stop and report. Don't look for a
workaround.

---

## Mode: cut

### 1. Decide the name and version string

Follow what the repo does **now**, not what an old doc says. List recent release branches
and read their version strings (the profile gives the command and the observed table):

```sh
git fetch <canonical> --prune
git branch -r | grep -E '<canonical>/release/' | sort -V | tail -15
```

If the kind the user wants has no recent precedent, show the documented form and the
closest current form, and ask. Leave other version variables alone unless told to.

### 2. Create the branch from the Release Version tip

```sh
git switch -c release/<next> --no-track <canonical>/release/<release>
```

With a clean working tree. Don't stash the user's work without asking.

### 3. Bump the version

Edit only the version lines the profile names, keeping trailing comments. Then:

```sh
git diff --stat   # exactly the expected file(s) and line(s)
git commit -am "<the profile's bump message>"
```

Keep any CI-skip marker the profile's message uses, so the bump doesn't run a pipeline.

### 4. ⚠ Push to the canonical repo

Release branches usually live on the canonical repo, not a fork; this is the one case where
pushing there is correct. Show the branch name, base sha, and diff, and get an explicit yes:

```sh
git push <canonical> release/<next>:release/<next>
```

Never force-push a release branch. If it already exists upstream, stop and report.

### Report

```
Next version: release/<next>  (from <canonical>/release/<release> @ <sha>)
Bump:         <VAR> '<old>' → '<new>'
Commit:       <sha> <subject>
Pushed:       yes → <canonical> | not pushed (awaiting confirmation)
```

---

## Mode: finalize

This is a long, multi-day checklist with human gates. Track progress with tasks. At the
start, ask for or confirm these and keep them in view:

```
Release Version: branch release/<rel>     Jira fixVersion: <…>
Next Version:    branch release/<next>    Jira fixVersion: <…>
```

Never derive a fixVersion from a branch name.

Follow the profile's checklist in order. The generic shape, with the skill or agent for
each part:

1. **Merge older active releases forward.** Find older release branches with commits the
   Release Version lacks (`git log --oneline <canonical>/release/<rel>..<canonical>/release/<old> | wc -l`),
   using the profile's definition of "active". Confirm the list, then use the **`mergeback`**
   skill for each hop, oldest first, merging each MR/PR before starting the next.
2. **Cut the Next Version branch.** Mode **cut** above (⚠ push).
3. **Sweep open MRs/PRs.** Run **`release-ticket-triage`** (section A). Draft the team
   heads-up listing the tickets whose target is about to change (give the user the text if
   there's no messaging tool). Then ⚠ **merge** the ones whose ticket is ready (only when the
   pipeline passed and no rebase is needed; otherwise list them for the user), ⚠ transition
   those tickets, and ⚠ **retarget** the rest to the Next Version branch while swapping
   their fixVersion. Tickets whose fixVersion doesn't contain the Release Version go to the
   profile's owner for that call; change nothing.
4. **Verify the pipeline** on the release branch. The latest must have succeeded. **Record
   the pipeline ID**; later steps need it. Draft the note asking the test team to run their
   baseline. (Optional: `release-regression` can run a regression on the build.)
5. **fixVersion cleanup.** `release-ticket-triage` section B. ⚠ Trash resolved-not-Done
   tickets and ⚠ move incomplete ones to the Next fixVersion. **Add before you remove**, so
   no ticket is ever left without a version.
6. **Release-notes prep.** `release-ticket-triage` section C. ⚠ Apply the approved title
   rewrites and internal-only labels, then draft the note telling the docs team.
7. **Gate: wait** for the profile's release gate (e.g. the test team's baseline with no
   high-severity bugs). Never pass it on your own judgement; ask the user.
8. **Artifact cleanup**, if the profile has one. Without access, give the user the exact
   keep/delete list. With access and a request, show the listing and the delete set first,
   and dry-run before the real run.
9. **Merge back** the release into the development branch with **`mergeback`**, and bring the
   Next Version up to date per the profile's rule.
10. **⚠ Tag** the commit the release shipped from (the tip the recorded pipeline built;
    check the sha matches), using the profile's tag pattern, and push the tag.
11. **⚠ Delete the release branch**, only after the tag is pushed and verified
    (`git ls-remote <canonical> refs/tags/<tag>`) and every mergeback from it has merged. Get
    a separate confirmation, and ask whether they want this at all.
12. **⚠ Mark the Jira release Released.** If there's no tool for it, give the user the
    release page link.

### Progress report

After each session, print the checklist with a status per step: ✅ done (with evidence:
MR/PR URL, sha, pipeline id), ⏳ waiting on <who>, ⬜ not started, ⏭ skipped (why). Always
carry the recorded pipeline ID forward.
