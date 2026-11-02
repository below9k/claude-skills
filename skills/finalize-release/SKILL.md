---
name: finalize-release
description: >
  Walk through finalizing a Savi sled release (e.g. release/v3.5.10): merge
  older active releases forward, cut the next release branch, sweep open MRs
  (merge or retarget), clean up Jira fixVersions, prep tickets for release
  notes, verify the pipeline, merge back to develop, tag sled/v<ver>, delete
  the release branch, and mark the Jira release Released. Use when the user
  says "finalize", "close out", "wrap up", or "ship" a release, or asks for
  any step of the release finalization checklist. Every outward-facing step
  needs explicit confirmation.
---

# Finalize a release

Source: *Finalizing a release (example v3.5.0)*, `oliveriq.atlassian.net/wiki/x/wJkiag`
(written 2024; deviations seen in today's repo are called out below). Glossary: **Release
Version** is the version being finalized. **Next Version** receives the unfinished MRs.

This is a long, multi-day checklist with human gates (System Test baseline, Documentation).
Track progress with tasks. At the start, ask for or confirm these four things and keep them in
view:

```
Release Version: branch release/v<rel>       Jira fixVersion: <e.g. 2026-Q4.1>
Next Version:    branch release/v<next>      Jira fixVersion: <…>
```

The Jira fixVersion names are quarter-based now, not `3.5.x`, so never derive one from the
other.

## Confirmation rules

The read-only steps (inspection, triage, drafting) need no confirmation. **Everything below
marked ⚠ is outward-facing or destructive.** Present exactly what will happen (the list of MRs
or tickets, the before → after, the command) and wait for an explicit yes **for that step**.
Approval for one step does not carry to the next. Batch where it helps: one yes per table, not
per row. If a tool returns 403 or something you did not expect, stop and report. Do not look
for a workaround.

## Steps

**1. Merge older active releases forward.** Find every active `release/v*` older than the
Release Version that has commits the Release Version lacks (`git log --oneline
base/release/v<rel>..base/release/v<old> | wc -l`). Treat a branch as active if it still exists
upstream and has no `sled/v<old>` tag, or if it is a still-patched hotfix line. Confirm the list
with the user. Then use the **`mergeback`** skill for each hop, oldest first, merging each MR
before starting the next.

**2–6. Cut the Next Version branch.** Use the **`cut-release-branch`** skill (⚠ push to
upstream).

**7–10. Sweep open MRs.** Run the **`release-ticket-triage`** agent (section A). Then:
- Draft the `#developers` Slack message listing the tickets whose target is about to change.
  There is no Slack tool here, so give the user the text to paste.
- ⚠ **Merge** the MRs whose ticket is *Ready to Merge* or *Done*. Use the GitLab MCP
  `merge_merge_request` only when its pipeline has passed and it needs no rebase. Otherwise list
  it for the user. After merging, ⚠ transition the ticket to *Done*.
- ⚠ **Retarget** the rest: `update_merge_request target_branch=release/v<next>`, and in Jira
  replace the Release fixVersion with the Next fixVersion (`editJiraIssue` with
  `update.fixVersions: [{add:{name:next}},{remove:{name:rel}}]`).
- If a ticket's fixVersion does not contain the Release Version, hand it to Jamie (product)
  and change nothing.

**11–13. Verify the pipeline.** `list_pipelines ref=release/v<rel>` on `savicontrols/savi`.
The latest pipeline must have succeeded. **Record the pipeline ID.** Steps 29–30 need it. Then
draft the note telling System Test to run baseline tests. (Optional: the `rc-regression` agent
can run a regression on the build.)

**14–23. Jira fixVersion cleanup.** Use `release-ticket-triage` section B. Then:
- ⚠ Resolution is set but is not Done → add `Trash`, then remove the Release fixVersion.
- ⚠ Not complete (To Do / In Progress) → add the Next fixVersion, then remove the Release
  fixVersion. **Add before you remove**, so no ticket is ever left without a version.
- Apply through `editJiraIssue` per ticket. The UI's Bulk Change is an equivalent the user can
  do by hand if there are many.

**24–27. Release-notes prep.** Use `release-ticket-triage` section C. ⚠ Apply the approved
title rewrites and `Internal` labels. Then draft the note telling Documentation they can write
the release notes.

**28. Gate: wait** until System Test reports the baseline is done with **no Severity A or B
bugs**. Do not go past this on your own judgement. Ask the user.

**29–30. Artifact cleanup** in the `savi-distro/savi` share on the QNAP NAS and in the
`savi-distro` S3 bucket. Remove the Release Version artifacts whose pipeline ID does not match
the one recorded in step 12. There is no NAS or S3 access configured here. Give the user the
exact list or prefix of what to keep and what to delete. If `aws` is configured and the user
asks you to do the S3 part, ⚠ show the `aws s3 ls` listing and the delete set first, and use
`--dryrun` before the real run.

**31. Mergeback to develop.** Use the **`mergeback`** skill: `release/v<rel>` → `develop`,
branch `SN-mergeback-<rel>-to-develop`.

**32. Bring the Next Version up to date.** If the minor version changes (3.5.x → 3.6.x), use the
`mergeback` skill to merge `develop` into `release/v<next>`. Otherwise, if the Release Version
changed after the Next Version branch was cut (`git log base/release/v<next>..base/release/v<rel>`),
use the `mergeback` skill for Release → Next.

**33. ⚠ Tag.** Tag the commit the release shipped from (the tip of `release/v<rel>` that the
recorded pipeline built: check that `git rev-parse base/release/v<rel>` matches the pipeline's
sha):
```sh
git tag -a sled/v<rel> <sha> -m "Sled release <rel>" && git push base sled/v<rel>
```
Existing tags are named like `sled/v3.5.6` and `sled/v3.5.1-hf1`.

**34. ⚠ Delete the upstream Release branch** (`git push base --delete release/v<rel>`). Only do
this after the tag is pushed and verified (`git ls-remote base refs/tags/sled/v<rel>`), and
after every mergeback from it has merged. Get a separate confirmation for this step. Recent
practice has kept some old release branches, so ask whether they want this at all.

**35. ⚠ Mark the Jira release Released.** There is no MCP tool for releasing a Jira version, so
give the user the link to the release page and ask them to click it.

## Progress report

After each session, print the checklist with a status for each step: ✅ done (with evidence:
MR/URL/sha/pipeline id), ⏳ waiting on <who>, ⬜ not started, ⏭ skipped (why). Always carry the
recorded pipeline ID forward.
