---
name: cut-release-branch
description: >
  Create the next Savi sled release branch (hotfix, rc, alpha, beta, or point
  release) from a release that is being finalized, and bump RELEASE_VER /
  DRIVERS_RELEASE_VER in .gitlab-ci.yml. Use when the user asks to "cut",
  "create", or "start" a release branch such as release/v3.5.10-rc1 or
  release/v3.5.8.1, or to bump the sled version. Pushing a release branch to
  savicontrols/savi is outward-facing and needs explicit confirmation.
---

# Cut the next release branch

This covers steps 2–6 of *Finalizing a release* (`oliveriq.atlassian.net/wiki/x/wJkiag`). The
*Release Version* is the one being finalized. The *Next Version* receives the unfinished MRs.

## 1. Decide the name and version string

The Confluence doc (2024) uses `-hf1` / `3.5.1.h1`. **Current branches do not.** Follow what the
repo does now, and check before you choose:

```sh
git fetch base --prune
git branch -r | grep -E 'base/release/v' | sort -V | tail -15
for b in $(git branch -r | grep -E 'base/release/v' | sort -V | tail -4); do
  echo "$b: $(git show $b:.gitlab-ci.yml | grep -E '^\s+(RELEASE_VER|DRIVERS_RELEASE_VER):' | tr -s ' ' | tr '\n' ' ')"; done
```

Observed 2026-10 (re-check):

| Kind | Branch | `RELEASE_VER` | `DRIVERS_RELEASE_VER` |
| --- | --- | --- | --- |
| point | `release/v3.5.10` | `'3.5.10'` | `'3.5.10'` |
| hotfix (4-part) | `release/v3.5.8.1` | `'3.5.8.1'` | `'3.5.8.1'` |
| release candidate | `release/v3.5.8-rc3` | `'3.5.8-rc3'` | `'3.5.8'` (the base version) |
| legacy hotfix | `release/v3.5.5-hf1` | `'3.5.5-hf1'` | — |
| alpha / beta (doc) | `release/v3.5.0-alpha1` / `-beta1` | doc says `3.5.0.a1` / `.b1`; none recent | — |

If the kind the user wants has no recent precedent, show them the doc's form and the closest
current form, and ask. Leave `SLED_MIN_UPGRADE_VER` alone unless told otherwise.

## 2. Create the branch from the Release Version tip

```sh
git switch -c release/v<next> --no-track base/release/v<release>
```

Do this with a clean working tree. Do not stash the user's work without asking.

## 3. Bump the version

Edit only the `RELEASE_VER` (and, per the table, `DRIVERS_RELEASE_VER`) lines in `.gitlab-ci.yml`.
Keep their trailing comments. Then:

```sh
git diff --stat   # exactly one file, one or two lines
git commit -am "[skip-ci] Updated sled release version to <next>"
```

Recent commits use `Updated sled release version to 3.5.10`; the doc uses `[skip-ci] Changed
sled version to …`. Either is fine. Keep `[skip-ci]` so the bump does not trigger a pipeline.

## 4. Push to upstream — confirm first

Release branches live on **savicontrols/savi** (`base`), not on the fork. That makes this the
one case where pushing to upstream is correct. It is still outward-facing: show the branch name,
the base sha, and the diff, then get an explicit yes before running:

```sh
git push base release/v<next>:release/v<next>
```

Never force-push a release branch. If it already exists upstream, stop and report.

## Report

```
Next version: release/v<next>  (from base/release/v<release> @ <sha>)
Bump:         RELEASE_VER '<old>' → '<new>'  [DRIVERS_RELEASE_VER …]
Commit:       <sha> <subject>
Pushed:       yes → base | not pushed (awaiting confirmation)
```
