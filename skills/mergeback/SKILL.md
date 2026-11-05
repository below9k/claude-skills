---
name: mergeback
description: >
  Perform a mergeback: merge an older release branch into a newer one
  (e.g. release/v3.5.6 into release/v3.5.8), a release branch into
  develop/main, or develop into the next release. Use whenever the user says
  "mergeback", "merge back", "merge 3.5.6 into 3.5.8", "carry fixes forward",
  or when finalizing a release needs older active releases merged forward.
  Produces the two-commit mergeback (a mechanical merge, then a single
  adaptations commit holding every judgement call), walks the repo profile's
  hotspots and migration rules, pushes per the profile, and opens the MR/PR.
---

# Mergeback

A **mergeback** merges an older version of the project into a newer one, so fixes made on
an older release aren't stranded. The method here keeps every judgement call reviewable:
one commit that is the merge itself, resolved mechanically, and one commit that holds every
place "target wins" was the wrong answer.

Load the repo profile (`~/.claude/repos/README.md`) and read the **Mergebacks** section of
its `release.md`: remotes, product lines, branch naming, the merge method, how to open the
MR/PR, the **hotspot table**, migration rules, and model commits to imitate. Without one,
ask about remotes, branch naming and migrations before step 3.

Steps 1 and 2 are read-only. Steps 3–7 create local commits. Step 8 is outward-facing and
follows the `push` skill's identity and remote rules. **Never push to a release branch or
the development branch directly.**

## 1. Work out what is being merged into what

Inputs: a **source** (older) and a **target** (newer) branch. If the user named versions,
map them to the canonical remote's release branches (`git remote -v` to find it).

```sh
git fetch <canonical> --prune
git branch -r | grep -E '<canonical>/release/' | sort -V
```

- **Within one product line.** Repos that ship several products from one tree (per the
  profile) merge back within a line only. If asked to merge across lines, stop and confirm.
- **Chained or direct.** With several active releases older than the target, default to a
  chain: one MR/PR per hop, each merged before the next starts. Direct is allowed. If the
  user didn't say, use the chain and list the hops first.
- **Special targets** (release finalization): release → development branch, and
  development → next release when the minor version changes. The profile says which.

## 2. Preflight: is there anything to merge, and what is the real merge base?

```sh
SRC=<canonical>/<source>; TGT=<canonical>/<target>
git merge-base $TGT $SRC                       # the real divergence point
git log --oneline --first-parent $TGT..$SRC     # what comes over
git log --oneline $TGT..$SRC | wc -l
git diff --stat $(git merge-base $TGT $SRC) $SRC | tail -1
```

- **Zero commits**: nothing to do. Say so and stop.
- **Check the merge base against what you expect.** Release branches get renamed and recut;
  a surprising merge base belongs in the description.
- Find the previous mergeback for this pair (`git log --merges --oneline -i --grep=mergeback $TGT`)
  and read its adaptations commit; the same hotspots usually come back.
- Flag **backports**: source commits that backport work the target already has. They should
  merge to the target's content and produce no diff.
- Look for **closed, unmerged** mergebacks into the same target. Name any in the
  description and reuse its adaptations commit as a reference. If you can't see why it was
  closed, say so; don't guess.
- When the target has diverged little, classify every file by diffing the **merged tree
  against the source**: every remaining difference should be in the target's own delta
  (`git diff --name-only $MB $TGT`). Files both sides changed are the only ones needing
  judgement.
- Work in a **worktree** (`git worktree add -b <branch> --no-track <tmp-path> $TGT`) so the
  user's checkout stays untouched. Run slow regeneration tools in the background.
- The working tree must be clean (`git status --porcelain`). If not, stop and ask. Don't
  stash the user's work on your own initiative.

## 3. Branch: cut from the target tip and merge the source in

```sh
git switch -c <mergeback-branch> --no-track $TGT
```

- Name it per the profile (e.g. `<PRJ>-mergeback-<src>-to-<tgt>`, `-r2` for a redo). This is
  usually a documented exception to the normal feature-branch format.
- `--no-track`, so a bare `git push` can't try to update a release branch.
- Always cut from the **target**. **If the target moves before the MR/PR merges, don't use a
  "rebase" button**: a rebase linearises the branch and throws away the merge commit, which
  is the whole record of the mergeback. Redo it instead: recut from the new target tip,
  repeat the mechanical merge, and cherry-pick the adaptations commit.

## 4. Commit 1: the mechanical merge

```sh
git merge --no-ff --no-commit $SRC
git diff --name-only --diff-filter=U                 # conflicted files
git status --short | grep -E '^(DU|UD|AU|UA|DD|AA)'  # conflicts -X won't settle
```

Resolve **every conflict toward the target**, with no judgement calls in this commit:

- Resolve at the hunk level. `git merge -X ours $SRC` is fine: it picks a side only for
  *conflicting* hunks and keeps cleanly auto-merged source hunks.
- **Never** `git merge -s ours` (discards the whole source) or `git checkout --ours <file>`
  (reverts the whole file, silently dropping the source's clean hunks in it).
- `-X` doesn't settle modify/delete, rename/rename or add/add conflicts. Take the target's
  side by hand and **write each one down**: that's where files get lost.
- The only exceptions taken from the source in this commit are ones the profile lists as
  inert. State them in the message.

Commit message (adjust the counts and the merge-base note):

```
Merge <source> into <target> (mechanical resolution)

This commit is the merge itself with every conflict resolved toward the
target branch. It contains no hand-made decisions, and its tree is
deliberately not guaranteed correct: <list what you already know is
wrong, e.g. a registration missing, lockfiles not regenerated>.

It exists so the follow-up commit isolates every judgement call for
review. See that commit, and the merge request description.

<merge-base note if it is surprising>
```

Before committing, confirm `git rev-parse MERGE_HEAD` equals `git rev-parse $SRC`. The
second parent has to be the source tip; it's the record of what was merged.

## 5. Commit 2: "Apply mergeback conflict resolutions and adaptations"

The **only** commit with judgement in it, and the one reviewers read. Go through every place
"target wins" was wrong, and where the source's change needs adapting to the target's
structure. **Walk the profile's hotspot table** and record what you found for each row,
even "nothing needed". Without a profile, check at least: version strings (usually the
target's value), registrations and manifests the source added to, lockfiles (regenerate;
never hand-merge), build and lint config lost to rename/delete conflicts, CI changes made on
both sides, migrations (§6), and backports.

Commit message: a lead paragraph ("Everything in the preceding merge commit was resolved
mechanically toward <target>. This commit contains every place that answer was wrong, and
is the only commit on this branch with a judgement call in it."), then one paragraph per
decision: *why*, then the files, indented two spaces. The profile may name model commits.

If there's nothing to adapt, say so in the description. Don't create an empty commit.

## 6. Migrations: check every mergeback

If the repo has ordered migrations (the profile says where, and how they're tracked), list
the ones added on each side since the merge base. If both sides added some, the prefixes
may interleave, and the side sorting **lower** gets silently skipped on upgrade from the
other line. Fix it with the profile's rule (typically rename if unreleased, forward-copy
if shipped; both re-apply the body, so check idempotency). If the repo has an ordering
checker, run it the way the profile says. If no migrations come over, say "clean, because
the source added none since `<mb>`", which is stronger than "the checker passed".

## 7. Verify before pushing

```sh
git merge-base --is-ancestor $SRC HEAD && git merge-base --is-ancestor $TGT HEAD && echo both-tips-in
git diff --stat $TGT...HEAD | tail -1
comm -23 <(git diff --name-only $TGT HEAD | sort) <(git diff --name-only $MB $SRC | sort)
```

- **Account for every file** in `git diff --name-only $TGT HEAD`: each comes from the source
  delta or is named in the adaptations commit. Anything else is a bug.
- Lint the touched packages with the profile's gates, and grep for leftover conflict
  markers (`git grep -n '^<<<<<<<\|^>>>>>>>'`).
- Run the **`mergeback-auditor`** agent on the branch. Fix what it confirms.
- Record honestly what wasn't run (full build, tests, a deploy).

## 8. Push and open the MR/PR

Follow `push` for identity and remotes, and the profile for how the MR/PR must be created
(some CIs only trigger when it's created a particular way). Description template:

```markdown
<one line: which mergeback, link to the previous one for this pair, N commits carried>

## How to review this
| | |
|---|---|
| `<sha1>` | The merge itself, resolved mechanically. |
| `<sha2>` | **Every judgement call.** This is the commit to review. |
<conflict count, which side each went to, any exception and why it is inert>

## Do not squash
`<sha1>` has `<source tip>` as its second parent; that is the record of the mergeback. The
branch is cut from the target tip, so a rebase-merge has nothing to replay. Leave squash off
and do not rebase — if the target moves, the mergeback will be redone.

## What comes over
Merge base `<mb>`. Tickets: … Backports that produce no diff: …

## What is in `<sha2>`
<one bold-led paragraph per decision>

## Migrations
<clean-and-why, or the interleave and its fix; how the checker was run>

## Verification
<what was run and passed> / Not run locally: …
```

Then make sure **squash is off**, add reviewers if the user names them, and link a ticket
only if the user gives one.

## Report

```
Mergeback: <source> → <target>   (hop i of n)
Branch:    <branch>  (cut from <target sha>)
Commits:   <sha1> mechanical (<N> conflicts → target)
           <sha2> adaptations (<k> decisions)  | none needed
Migrations: <clean because… | fixed: … | n/a>
Verified:  <list>   Not run: <list>
MR/PR:     <url>
Next:      <next hop, or "merge this before starting <next>">
```
