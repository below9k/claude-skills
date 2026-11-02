---
name: mergeback
description: >
  Perform a Savi mergeback: merge an older release branch into a newer one
  (e.g. release/v3.5.6 into release/v3.5.8), a release branch into develop,
  or develop into the next release. Use whenever the user says "mergeback",
  "merge back", "merge 3.5.6 into 3.5.8", "carry fixes forward", or when
  finalizing a release needs older active releases merged forward. Produces
  the house-style two-commit mergeback (mechanical merge, then a single
  adaptations commit holding every judgement call), checks migration
  ordering, pushes to the user's fork, and opens the MR. Repos live in
  ~/develop/savi*.
---

# Mergeback

A **mergeback** merges an older version of the project into a newer one, so fixes made on an
older release are not stranded. Sources: the Confluence pages *Finalizing a release (example
v3.5.0)* (`oliveriq.atlassian.net/wiki/x/wJkiag`) and *How the SAVI Monorepo Actually Works*
(`/wiki/x/AQAYcQ`), plus the house style of Paul Ward's mergebacks (!4901 and !4930), which is
what reviewers now expect.

Steps 1 and 2 are read-only. Steps 3–7 create local commits. Step 8 is outward-facing: push and
MR creation follow the `gitlab-push` skill's identity and remote rules. **Never push to a
`release/*` branch or `develop` directly.**

---

## 1. Work out what is being merged into what

Inputs: a **source** (older) and a **target** (newer) branch. If the user named only versions,
map them to `base/release/v<ver>`. Remotes in `~/develop/savi`: `base` = savicontrols/savi
(upstream), `1fork` = the user's fork. Check with `git remote -v` in other clones.

```sh
git fetch base --prune
git branch -r | grep -E 'base/release/v' | sort -V
```

**Mergebacks run within one product line.** `release/v*` (sled software) is one line.
`release/sled-fw/*`, `release/cloud/*`, `release/dsp/*`, `release/eed/*`, `release/snt/*`,
`release/drivers/*` are separate products that share the repo. Treat each one as its own repo.
A sled release consumes the firmware *artifact*, so firmware source that looks stale on
`release/v*` is normal and not a reason to merge across lines. If the user asks for a merge
across product lines, stop and confirm it is really what they want.

**Chained or direct.** When several active releases are older than the target, the default is a
chain, one MR per hop, each merged before the next starts (3.4.0-hf6 → 3.4.1-hf3 → 3.4.2-hf1 →
3.5.0). Merging each one straight into the target is also allowed. If the user did not say, use
the chain and list the hops before starting.

**Special targets** (finalize-release steps 31–32):
- Release → `develop`: branch `SN-mergeback-<release>-to-develop` (or `SN-mergeback-<release>`).
- If the minor version changes between the release and the next version (3.5.x → 3.6.x),
  **develop** is merged into the next version. Otherwise the release is merged into the next
  version, and only when the release changed after the next-version branch was cut.

## 2. Preflight — is there anything to merge, and what is the real merge base?

```sh
SRC=base/release/v3.5.6; TGT=base/release/v3.5.8
git merge-base $TGT $SRC                       # the real divergence point
git log --oneline --first-parent $TGT..$SRC     # what comes over
git log --oneline $TGT..$SRC | wc -l
git diff --stat $(git merge-base $TGT $SRC) $SRC | tail -1
```

- **Zero commits** means nothing to do. Say so and stop.
- **Check the merge base against what you expect.** Release branches here have been renamed and
  recut. For example, `release/v3.5.6` was reset to the v3.5.5.1 tip and rebuilt from
  backports, so 3.5.6 → 3.5.8 had a merge base on 3.5.5 and 51 unmerged commits.
  `cli-bazel/closest-release` explains the history. A surprising merge base belongs in the MR
  description.
- Find the previous mergeback for this pair (`git log --merges --oneline -i --grep=mergeback $TGT`). Read its adaptations commit, because the same hotspots usually come
  back.
- Flag **backports**: source commits that backport work the target already has (e.g. SN-6069
  backports SN-5895). They should merge to the target's content and produce no diff.
- Check for **closed, unmerged** mergebacks into the same target (`list_merge_requests
  target_branch=<tgt> state=all`). For example, !4931 (3.5.6 → develop) was closed on
  2026-09-09 and is superseded by any later mergeback whose source contains that tip. Name it
  in the MR description and reuse its adaptations commit as a reference. The MCP token cannot
  read MR notes (403), so you cannot see why it was closed. Say that, and do not guess.
- When the target has diverged little (develop vs the newest release), classify every file by
  diffing the **merged tree against the source**. Every remaining difference should be a file
  in the target's own delta (`git diff --name-only $MB $TGT`). Files both sides changed are the
  only ones that need judgement.
- Do the work in a **worktree** (`git worktree add -b <branch> --no-track
  $CLAUDE_JOB_DIR/tmp/<name> $TGT`) so the user's checkout and branch stay untouched.
  `./cli-bazel/refs` runs there with a cold bazel output base, which is slow, so run it in the
  background.
- Make sure the working tree is clean (`git status --porcelain`). If it is not, stop and ask.
  Do not stash the user's work on your own initiative.

## 3. Branch — cut from the target tip and merge the source in

```sh
git switch -c SN-mergeback-3.5.6-to-3.5.8 --no-track $TGT
```

- Naming: `SN-mergeback-<sourceVer>-to-<targetVer>`, using the version without `release/v`
  (`SN-mergeback-3.4.2-hf1-to-3.5.0`). The Confluence doc says the `SN-` prefix makes CI build
  the full set of sled artifacts. Today's `.gitlab-ci.yml` keys the artifact jobs on the MR
  target matching `release/v*`, so the prefix is a convention more than a requirement. Keep it
  anyway. If the user wants a re-do of an existing mergeback, add `-r2`.
- This is the documented exception to `gitlab-push`'s `PRJ-####/PRJ-###-desc` branch format.
- Use `--no-track` so a bare `git push` cannot try to update the fork's release branch.
- Always cut from the **target**. The project's merge method is rebase-merge, and a target-cut
  branch leaves GitLab nothing to replay. **If the target moves before the MR merges, do not
  press GitLab's Rebase button.** A rebase linearises the branch and throws away the merge
  commit, which is the whole record of the mergeback. Redo it instead: recut from the new
  target tip, repeat the mechanical merge, and `git cherry-pick` the adaptations commit.

## 4. Commit 1 — the mechanical merge

```sh
git merge --no-ff --no-commit $SRC
git diff --name-only --diff-filter=U            # conflicted files
git status --short | grep -E '^(DU|UD|AU|UA|DD|AA)' # delete/rename conflicts -X won't settle
```

Resolve **every conflict toward the target**, with no judgement calls in this commit:

- Resolve at the hunk level. `git merge -X ours $SRC` is fine for this, because it only picks a
  side for *conflicting* hunks and keeps cleanly auto-merged source hunks.
- **Never** use `git merge -s ours` (it discards the entire source) or
  `git checkout --ours <file>` (it reverts the whole file and silently drops the source's clean
  hunks in that file).
- `-X` does not settle modify/delete, rename/rename, or add/add conflicts. Take the target's
  side by hand, and **write each one down**, because these are where files get lost (e.g.
  `driver-everpass/.eslintrc.js` was lost to a rename/rename against an unrelated package's
  boilerplate).
- Exception you may take from the source in this commit, and must state: the `@savi/`
  `integrity` hashes in `app/package-lock.json` have no effect either way, because
  `sled/deployment/build-app.sh` strips them via `//devops-tools/remove-lock-integrity` before
  install.

Commit message (adjust the counts and the merge-base note):

```
Merge release/v3.5.6 into release/v3.5.8 (mechanical resolution)

This commit is the merge itself with every conflict resolved toward the
target branch. It contains no hand-made decisions, and its tree is
deliberately not guaranteed correct: <list what you already know is
wrong, e.g. driver X unregistered, lockfiles not regenerated>.

It exists so the follow-up commit isolates every judgement call for
review. See that commit, and the merge request description.

<merge-base note if it is surprising>
```

Before committing, confirm `git rev-parse MERGE_HEAD` equals `git rev-parse $SRC`. The second
parent has to be the source tip. That parent is the record that this version of the source was
merged.

## 5. Commit 2 — "Apply mergeback conflict resolutions and adaptations"

This is the **only** commit with judgement in it, and it is the commit reviewers read. Go through
every place where "target wins" was the wrong answer, and where the source's change needs
adapting to the target's structure. Check each hotspot below and record what you found, even
when the answer is "nothing needed":

| Hotspot | What goes wrong | Fix |
| --- | --- | --- |
| `.gitlab-ci.yml` `RELEASE_VER`, `DRIVERS_RELEASE_VER` | source version leaks in | release → release: must stay at the **target's** value. Release → `develop`: develop tracks the newest release merged into it (it got `3.5.6` that way), so the source's value is right. State it either way |
| `.gitlab-ci.yml` `SLED_INCLUDE_FW`, `SLED_FIRMWARE_VER` | a firmware bump on the source loses a conflict to an adjacent target-only line (e.g. develop's `CLOUD_VER`) | take the source's firmware bump; leave the other product versions at the target's values |
| Driver registration | 3.5.8+ deleted `savi-drivers.yaml` and moved the list to `lerna/packages/sled/drivers/` | source drivers added to the yaml become entries in `sled/drivers/index.js` and `sled/drivers/package.json` (`workspace:*`); otherwise the driver merges in as dead code |
| `savi.yaml` and `savi-prod.yaml` | a service added on one side only | both files must agree (root `CLAUDE.md`) |
| `lerna/packages/pnpm-lock.yaml`, `MODULE.bazel.lock` | hand-merged lockfiles | regenerate with `./cli-bazel/refs`; never hand-merge. Lockfile is v6 (pnpm 8), so do not let pnpm 10 rewrite it |
| `pnpm-workspace.yaml`, `.bazelignore` | `refs` strips the manual `bazel-*` entries | confirm the begin/end marker block survived |
| Per-package `.eslintrc.js`, `BUILD.bazel` | lost to rename/delete conflicts | restore from the source |
| `.bazelrc`, `.bazelrc.d/*.bazelrc` | generated preset files, flags that only make sense in pairs | keep the pair consistent; presets must match the generator |
| CI / `deployment/*.sh` | each side improved the pipeline separately | pick deliberately, explain why |
| Migrations (`lerna/packages/sled-core/migrations/scripts/`) | interleave trap: a migration skipped silently | §6, always |
| Backports | double application or a conflict with the target's original | should produce no diff; if they do, explain why |

Commit message: a lead paragraph ("Everything in the preceding merge commit was resolved
mechanically toward <target>. This commit contains every place that answer was wrong, and is the
only commit on this branch with a judgement call in it."), then one paragraph per decision. Say
*why*, then list the files indented by two spaces. See `git show 0fa9fa4d8` and
`git show cf41f1226` for the model.

If there is nothing to adapt, say so explicitly in the MR description. Do not create an empty
commit.

## 6. Migrations — check every mergeback

Read `lerna/packages/sled-core/migrations/CLAUDE.md` first. Short version: the filename is the
id, the DB keeps only a `lastRun` high-water mark, and anything that sorts before `lastRun` is
assumed to have run. A mergeback can turn a correct install refusal into a **silent skip**
(SN-6052).

```sh
MB=$(git merge-base $TGT $SRC)
git diff --name-only --diff-filter=A $MB $SRC -- lerna/packages/sled-core/migrations/scripts/   # coming over
git diff --name-only --diff-filter=A $MB $TGT -- lerna/packages/sled-core/migrations/scripts/   # target-only
ls lerna/packages/sled-core/migrations/scripts/ | sort | tail -5
```

- If both lists are non-empty, the prefixes may interleave. The side whose migration sorts
  **lower** loses: a sled arriving from the other line skips it. Fix it with the rule in that
  guide. **Rename** the losing file past the newest migration on the target if that line is
  unreleased, or **forward-copy** it if it has shipped. Both re-apply the body, so check that it
  is idempotent. A rename needs a remediation note for internal sleds.
- The checker is manual (its CI job is commented out, SN-5549). **Run it from a detached source
  tip, not from the mergeback branch.** From a target-cut branch, `merge-base HEAD <target>` is
  the target tip, so it applies the ordinary rule and prints SUCCESS having checked nothing
  (SN-6053):

  ```sh
  git switch --detach $SRC
  CI_MERGE_REQUEST_TARGET_BRANCH_NAME=${TGT#base/} CI_PROJECT_DIR=$(git rev-parse --show-toplevel) \
    node lerna/packages/devops-tools/check-sled-migrations/index.js --remote base
  git switch -
  ```

- **Known false positive:** the checker reads `git diff --name-only`, so a migration that was
  *modified* on the source (e.g. SN-5962 editing `1643150240299_equipment-closet-location.js`)
  gets treated as new and fails with `MigrationOrderError`. Confirm with `git diff
  --name-status $MB $SRC -- <file>`. An `M` on an id the target already has is not an
  ordering problem. Note it in the MR. Editing an already-applied migration only affects fresh
  databases, which is a separate question for the author of that change.
- If no migrations come over, say so in the MR ("clean, because the source added none since
  `<mb>`"). That is a stronger statement than "the checker passed".
- A wiped database proves nothing. Validation needs a populated DB whose `lastRun` matches the
  upgrade path.

## 7. Verify before pushing

```sh
git merge-base --is-ancestor $SRC HEAD && git merge-base --is-ancestor $TGT HEAD && echo both-tips-in
git diff --stat $TGT...HEAD | tail -1          # branch delta vs target
comm -23 <(git diff --name-only $TGT HEAD | sort) <(git diff --name-only $MB $SRC | sort)   # files not from the source delta
```

- **Account for every file** in `git diff --name-only $TGT HEAD`. Each one should come from the
  source delta (`git diff --name-only $MB $SRC`) or be named in the adaptations commit. Anything
  else is a bug.
- Lint the touched bazel packages: `./cli-bazel/lint //<pkg>`.
- Run `git grep -n '^<<<<<<<\|^>>>>>>>' -- lerna deployment` to catch leftover conflict markers.
- Run the **`mergeback-auditor`** agent on the branch. It checks the two-commit discipline, the
  hotspot table, and migrations independently. Fix what it confirms.
- Record honestly what was **not** run: `bazel build //...`, unit tests, a sled deploy. CI
  covers the first two.

## 8. Push and open the MR

Follow `gitlab-push` for identity. Create the MR through push options, not the MCP, so it is
authored by the user and CI triggers:

```sh
git push -u 1fork SN-mergeback-3.5.6-to-3.5.8:SN-mergeback-3.5.6-to-3.5.8 \
  -o merge_request.create -o merge_request.target_project=savicontrols/savi \
  -o merge_request.target=release/v3.5.8 \
  -o merge_request.title="Mergeback release/v3.5.6 into release/v3.5.8" \
  -o merge_request.description="Mergeback; full description follows" \
  -o merge_request.remove_source_branch
```

Push options reject newlines. Set the full markdown description afterwards with the GitLab MCP
`update_merge_request`. Description template (from !4930):

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
branch is cut from the target tip, so rebase-merge has nothing to replay. Leave squash off and do
not rebase — if the target moves, the mergeback will be redone.

## What comes over
Merge base `<mb>`. Tickets: … Backports that produce no diff: …

## What is in `<sha2>`
<one bold-led paragraph per decision>

## Migrations
<clean-and-why, or the interleave and its fix; how the checker was run>

## Verification
<what was run and passed> / Not run locally: …
```

Then:
- Make sure **squash is off** on the MR (`update_merge_request squash=false`).
- Add reviewers if the user names them.
- No Jira link is needed unless the user gives a ticket. Mergebacks usually have none.

## Report

```
Mergeback: <source> → <target>   (hop i of n)
Branch:    SN-mergeback-…  (cut from <target sha>)
Commits:   <sha1> mechanical (<N> conflicts → target)
           <sha2> adaptations (<k> decisions)  | none needed
Migrations: <clean because… | fixed: …>
Verified:  <list>   Not run: <list>
MR:        <url>
Next:      <next hop, or "merge this before starting <next>">
```
