# Savi release process

Read by `release`, `mergeback`, `mergeback-auditor`, `release-ticket-triage` and
`release-regression`. Source: the Confluence pages *Finalizing a release (example v3.5.0)*
(`oliveriq.atlassian.net/wiki/x/wJkiag`, written 2024) and *How the SAVI Monorepo Actually
Works* (`/wiki/x/AQAYcQ`), plus the house style of Paul Ward's mergebacks (!4901, !4930),
which reviewers now expect. Where the doc and today's repo differ, the repo wins; the
differences seen so far are noted below.

## Lines, branches and tags

- `release/v*` is the sled software line. `release/sled-fw/*`, `release/cloud/*`,
  `release/dsp/*`, `release/eed/*`, `release/snt/*` and `release/drivers/*` are separate
  products sharing the repo; treat each as its own repo. A sled release consumes the
  firmware *artifact*, so firmware source that looks stale on `release/v*` is normal.
- Release branches live on the canonical repo (`base` in `~/develop/savi`), not the fork.
- Shipped tags: `sled/v<ver>` (e.g. `sled/v3.5.6`, `sled/v3.5.1-hf1`). The latest
  shipped release is `git tag -l 'sled/v*' --sort=-v:refname | head -1`.
- Versions aren't contiguous: 3.5.8 follows 3.5.6.1; there is no 3.5.7. Branches have been
  renamed and recut: `release/v3.5.6` was reset to the v3.5.5.1 tip and rebuilt from
  backports. `cli-bazel/closest-release` explains the history.
- People: product owner for fixVersion questions is **Jamie**. Release heads-ups go to
  `#developers` on Slack (no Slack tool here; draft the text). System Test runs the baseline;
  Documentation writes release notes.
- Jira statuses that mean "ship it": *Ready to Merge*, *Done*.

## Version bump (cutting a branch)

Edit only `RELEASE_VER` (and per the table, `DRIVERS_RELEASE_VER`) in `.gitlab-ci.yml`,
keeping trailing comments. Leave `SLED_MIN_UPGRADE_VER` alone unless told. Check what recent
branches do before choosing a form:

```sh
git branch -r | grep -E 'base/release/v' | sort -V | tail -15
for b in $(git branch -r | grep -E 'base/release/v' | sort -V | tail -4); do
  echo "$b: $(git show $b:.gitlab-ci.yml | grep -E '^\s+(RELEASE_VER|DRIVERS_RELEASE_VER):' | tr -s ' ' | tr '\n' ' ')"; done
```

Observed 2026-10:

| Kind | Branch | `RELEASE_VER` | `DRIVERS_RELEASE_VER` |
| --- | --- | --- | --- |
| point | `release/v3.5.10` | `'3.5.10'` | `'3.5.10'` |
| hotfix (4-part) | `release/v3.5.8.1` | `'3.5.8.1'` | `'3.5.8.1'` |
| release candidate | `release/v3.5.8-rc3` | `'3.5.8-rc3'` | `'3.5.8'` (the base version) |
| legacy hotfix | `release/v3.5.5-hf1` | `'3.5.5-hf1'` | — |
| alpha / beta (doc only) | `release/v3.5.0-alpha1` / `-beta1` | doc says `3.5.0.a1` / `.b1` | — |

The doc's `-hf1` / `3.5.1.h1` forms are no longer used. Commit:
`[skip-ci] Updated sled release version to <next>` (`[skip-ci]` keeps the bump from
running a pipeline). Push: `git push base release/v<next>:release/v<next>`.

## Finalization checklist (doc step numbers)

| Steps | What | Savi specifics |
| --- | --- | --- |
| 1 | merge older active releases forward | active = exists upstream with no `sled/v<old>` tag, or a still-patched hotfix line |
| 2–6 | cut the Next Version branch | version bump above |
| 7–10 | sweep open MRs | `release-ticket-triage` A; Slack heads-up to `#developers`; merge with GitLab MCP `merge_merge_request` only when the pipeline passed and no rebase is needed; retarget with `update_merge_request target_branch=…`; fixVersion swap with `editJiraIssue update.fixVersions: [{add:{name:next}},{remove:{name:rel}}]`; a ticket whose fixVersion lacks the Release Version goes to Jamie |
| 11–13 | verify the pipeline | `list_pipelines ref=release/v<rel>`; **record the pipeline ID** (steps 29–30 need it); tell System Test to run the baseline |
| 14–23 | fixVersion cleanup | `release-ticket-triage` B. Resolution set but not Done → add `Trash`, remove Release. Not complete → add Next, remove Release. **Add before remove.** |
| 24–27 | release-notes prep | `release-ticket-triage` C; apply title rewrites and `Internal` labels; tell Documentation |
| 28 | **gate** | System Test reports the baseline done with **no Severity A or B bugs**. Ask the user; never judge this yourself |
| 29–30 | artifact cleanup | `savi-distro/savi` share on the QNAP NAS and the `savi-distro` S3 bucket: remove Release artifacts whose pipeline ID ≠ the recorded one. No NAS/S3 access here: give the user the keep/delete list. If `aws` is configured and asked, show `aws s3 ls` and use `--dryrun` first |
| 31 | mergeback to `develop` | branch `SN-mergeback-<rel>-to-develop` |
| 32 | bring Next up to date | minor version changes (3.5.x → 3.6.x): merge `develop` into Next. Otherwise Release → Next, only if Release changed after Next was cut |
| 33 | tag | `git tag -a sled/v<rel> <sha> -m "Sled release <rel>" && git push base sled/v<rel>`; `<sha>` is the tip the recorded pipeline built |
| 34 | delete the Release branch | `git push base --delete release/v<rel>` only after the tag is verified (`git ls-remote base refs/tags/sled/v<rel>`) and every mergeback from it merged. Recent practice kept some old branches: ask whether to do it at all |
| 35 | mark the Jira release Released | no MCP tool: give the user the release page link |

## Release-ticket triage constants

- GitLab `savicontrols/savi`; Jira cloudId `729cdf11-9f14-49de-9b13-464494ee2751`; projects
  on release branches: `SN`, `GAL`, `CLD` (check commit subjects for others).
- Shipped-range tickets:
  `git log --oneline --merges $PREV..base/<release> | grep -oE '\b(SN|GAL|CLD)-[0-9]+' | sort -u`
- **Internal** label when (a) a bug introduced and fixed within this release, never shipped
  (`git merge-base --is-ancestor <introducing sha> $PREV`), or (b) invisible to users and
  dealers (devops, CI, dev environment, tests, docs-only). Never propose removing it.
- Titles describe what changed, not the symptom ("Driver XYZ now correctly sends disconnect
  commands", not "Driver XYZ does not always send disconnect commands").

## Mergebacks

- Remotes in `~/develop/savi`: `base` = canonical, `1fork` = the user's fork.
- Branch `SN-mergeback-<srcVer>-to-<tgtVer>` (version without `release/v`, e.g.
  `SN-mergeback-3.4.2-hf1-to-3.5.0`; `-r2` for a redo). The doc says the `SN-` prefix makes
  CI build the full sled artifact set; today `.gitlab-ci.yml` keys those jobs on the MR target
  matching `release/v*`, so it's convention more than requirement. Keep it.
- Default is a **chain**, one MR per hop, each merged before the next (3.4.0-hf6 → 3.4.1-hf3
  → 3.4.2-hf1 → 3.5.0).
- The project's merge method is **rebase-merge**: never press GitLab's Rebase button on a
  mergeback; it linearises away the merge commit. Redo instead.
- `./cli-bazel/refs` regenerates lockfiles; in a fresh worktree it runs with a cold bazel
  output base, so run it in the background.
- Open the MR with push options so it's authored by the user and CI triggers (push options
  reject newlines; set the full description afterwards with `update_merge_request`):

  ```sh
  git push -u 1fork <branch>:<branch> \
    -o merge_request.create -o merge_request.target_project=savicontrols/savi \
    -o merge_request.target=release/v<tgt> \
    -o merge_request.title="Mergeback release/v<src> into release/v<tgt>" \
    -o merge_request.description="Mergeback; full description follows" \
    -o merge_request.remove_source_branch
  ```
- Model adaptations commits: `git show 0fa9fa4d8`, `git show cf41f1226`. !4931 (3.5.6 →
  develop) was closed unmerged on 2026-09-09; a later mergeback whose source contains that
  tip supersedes it.
- Inert exception allowed in the mechanical commit (state it): the `@savi/` `integrity`
  hashes in `app/package-lock.json`, which `sled/deployment/build-app.sh` strips via
  `//devops-tools/remove-lock-integrity` before install.

### Hotspots (check each in the adaptations commit, even when the answer is "nothing")

| Hotspot | What goes wrong | Fix |
| --- | --- | --- |
| `.gitlab-ci.yml` `RELEASE_VER`, `DRIVERS_RELEASE_VER` | source version leaks in | release → release: keep the **target's** value. Release → `develop`: develop tracks the newest release merged into it, so the source's value is right. State it |
| `.gitlab-ci.yml` `SLED_INCLUDE_FW`, `SLED_FIRMWARE_VER` | a source firmware bump loses to an adjacent target-only line (e.g. develop's `CLOUD_VER`) | take the source's firmware bump; other product versions stay at the target's |
| Driver registration | 3.5.8+ deleted `savi-drivers.yaml`; the list moved to `lerna/packages/sled/drivers/` | source drivers added to the yaml become entries in `sled/drivers/index.js` and `sled/drivers/package.json` (`workspace:*`), or they merge in as dead code |
| `savi.yaml` and `savi-prod.yaml` | a service added on one side only | both must agree (root `CLAUDE.md`) |
| `lerna/packages/pnpm-lock.yaml`, `MODULE.bazel.lock` | hand-merged lockfiles | regenerate with `./cli-bazel/refs`. Lockfile v6 (pnpm 8): don't let pnpm 10 rewrite it |
| `pnpm-workspace.yaml`, `.bazelignore` | `refs` strips the manual `bazel-*` entries | confirm the begin/end marker block survived |
| per-package `.eslintrc.js`, `BUILD.bazel` | lost to rename/delete conflicts (e.g. `driver-everpass/.eslintrc.js` lost to a rename/rename) | restore from the source |
| `.bazelrc`, `.bazelrc.d/*.bazelrc` | generated presets; flags that only make sense in pairs | keep pairs consistent; presets match the generator |
| CI / `deployment/*.sh` | each side improved the pipeline separately | pick deliberately, explain why |
| migrations | interleave trap | below, always |
| backports (e.g. SN-6069 backports SN-5895) | double application, or a conflict with the target's original | should produce no diff; explain if they do |

Auditor calibration: files `-X ours` leaves unmerged (modify/delete, e.g. `savi-drivers.yaml`,
which the target deleted) show the source's content in a reproduced merge; that's expected
if the target's deletion was kept. Lockfile diffs are low severity if commit 2 regenerates
them. New source packages need `package.json`, `BUILD.bazel`, `.eslintrc.js` and a
`pnpm-workspace.yaml` entry. Conflict-marker check:
`git grep -nE '^(<<<<<<<|>>>>>>>|=======$)' <branch> -- lerna deployment .gitlab-ci.yml`.

### Migrations

Read `lerna/packages/sled-core/migrations/CLAUDE.md` first. The filename is the id; the DB
keeps only a `lastRun` high-water mark; anything sorting before `lastRun` is assumed run. A
mergeback can turn a correct install refusal into a **silent skip** (SN-6052).

```sh
D=lerna/packages/sled-core/migrations/scripts
git diff --name-only --diff-filter=A $MB $SRC -- $D    # coming over
git diff --name-only --diff-filter=A $MB $TGT -- $D    # target-only
ls $D | sort | tail -5
```

- Both non-empty → prefixes may interleave; the side sorting **lower** loses (a sled from
  the other line skips it). **Rename** past the target's newest if that line is unreleased,
  **forward-copy** if it has shipped. Both re-apply the body: check idempotency (no
  `$inc`/`$push`/`$mul`/`$pop`/`$bit`, no insert without `_id`/upsert). A rename needs a
  remediation note for internal sleds.
- The checker is manual (its CI job is commented out, SN-5549). **Run it from a detached
  source tip**; from a target-cut branch it prints SUCCESS having checked nothing (SN-6053):

  ```sh
  git switch --detach $SRC
  CI_MERGE_REQUEST_TARGET_BRANCH_NAME=${TGT#base/} CI_PROJECT_DIR=$(git rev-parse --show-toplevel) \
    node lerna/packages/devops-tools/check-sled-migrations/index.js --remote base
  git switch -
  ```
- Known false positive: a migration *modified* on the source (e.g. SN-5962 editing
  `1643150240299_equipment-closet-location.js`) is treated as new and fails with
  `MigrationOrderError`. Confirm with `git diff --name-status $MB $SRC -- <file>`; an `M` on
  an id the target has isn't an ordering problem.
- None coming over → say "clean, because the source added none since `<mb>`", which is
  stronger than "the checker passed". A wiped DB proves nothing.

## Release-candidate regression

- RC branch `release/v3.5.8-rc1` belongs to line `release/v3.5.8`. Baseline = highest
  shipped `release/vX.Y.Z`, `.N` or `-hfN` below it, skipping `-rcN` and ad-hoc branches.
- Ticket keys in the range: `SN-`, `DSP-`, `GAL-`, `SLED-`.
- Artifacts and deploy: `qa-env.md`. Code tests: `testing.md` (no improvised runners).
