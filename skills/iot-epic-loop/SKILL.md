---
name: iot-epic-loop
description: One iteration of the living workflow for an MFB epic in FarmBot's `iot` repo (default MFB-1587, the DPS rewrite). Syncs PR and Jira state, triages new review waves, closes out and restacks after merges, keeps tickets accurate, and starts the next ready ticket within a WIP cap. Designed to run under `/loop /iot-epic-loop MFB-1587`; stops cleanly when context runs low.
effort: high
---

# Epic loop (one iteration)

Epic: $ARGUMENTS (if empty, use the **Epic** in `state.md`)

Contract: `~/.claude/skills/iot-ticket-write/ticket-standard.md`. Repo: your
checkout of `iot` (GitHub `farmbot-monitoring-solutions/iot`). Jira:
`farmbot.atlassian.net`, project MFB. State file: `state.md` next to this file. Read it
first and rewrite it last.

Standing authority (user, 2026-11-02): edit MFB tickets and existing DPS PRs (bodies,
labels, links, comments, follow-up commits on the PR's own branch). **Still ask first** for:
opening a new PR, merging, force-pushing a branch that someone else has reviewed, replying
in a teammate's review thread, closing or transitioning a ticket to Done, and anything
outside the epic.

## Starting, stopping and resuming

The user starts and stops this loop by hand, often because of token usage: a session runs
out, `/loop` gets cancelled, or a new session starts days later. **Nothing about one run
lives in this file.** Everything that changes between runs goes in `state.md`, so this skill
never needs editing on a rerun. If you notice yourself wanting to edit SKILL.md to record
progress, a decision or a new PR, put it in `state.md` instead. Edit this file only when
the *process* changes, and say so to the user.

**Starting** (`/loop /iot-epic-loop` or `/iot-epic-loop`, possibly in a fresh session with
no memory of earlier runs):

1. Read `state.md`. If the epic argument is missing, use `state.md`'s epic.
2. Check the **In flight** section. If it names work, finish that before anything else.
3. Recover from a hard stop, because the last run may have died mid-step without writing
   state. Check for:
   - uncommitted changes, or local commits not on the remote
     (`git -C <iot checkout> status -sb` and `git log @{u}..` on each DPS branch);
   - a ticket whose body has an epic-review note dated after the watermark but no matching
     line in **Done**;
   - PR bodies that disagree with their branch.

   Finish or roll back what you find, then log it in **Done**.
4. Steps 1–6 below must be safe to repeat. Before any write, check whether it is already
   done: is the link already there, the comment already posted, the commit already pushed?
   Re-running a step must never duplicate a comment, a link or a commit.

**Stopping on purpose** (context past ~70%, the user says stop, or nothing left to do):

1. Get to a clean boundary. Commit and push WIP on the PR's own branch (prefix the commit
   with `wip:` if the work isn't done), or stash it and record the stash name. Never leave a
   half-written Jira body.
2. Write `state.md`: a new watermark (advance it only past reviews and comments that are
   triaged or recorded in **In flight** / **Open PRs**; never past one you haven't read), and the **In flight** section with the ticket or PR,
   branch, step number, the exact next action, and anything you learned that isn't written
   down anywhere else.
3. End the loop with `ScheduleWakeup stop`, and tell the user in one line what to type to
   resume (normally just `/loop /iot-epic-loop`).

**Being stopped without warning:** this is why rule 4 matters. Keep each unit of work
small enough that losing it costs one step, and update **In flight** *before* starting a
long step (a ticket implementation or a review-fix wave), not only after.

## 0. Budget guard

Check context at the start of each step. Past ~70%, don't start a new step: do "Stopping on
purpose". Never start a ticket you can't finish in the budget left; record it as next in
**In flight** instead.

## 1. Sync (cheap, every iteration)

- `gh pr list --search "head:feat/mfb- OR head:docs/mfb-" --state all --json number,title,state,baseRefName,headRefName,reviewDecision,statusCheckRollup,updatedAt`
- For each open PR: CI status, `reviews` and inline comments **newer than `state.md`'s
  watermark**, and commits behind its base.
- Jira: `parent = <epic>` with status, labels and links (compact view; no descriptions
  unless needed).
- Assignee check (every iteration): `parent = <epic> AND status != BACKLOG AND status !=
  Done AND assignee is EMPTY`. Assign each hit to the epic's owner: the current
  Atlassian user (`atlassianUserInfo` gives the accountId). This covers Selected for Development as well as In Progress and PR REVIEW.
  Whenever the loop moves a ticket out of BACKLOG itself, assign it in the same step.
- Diff against `state.md`. If nothing changed, skip to step 5.

## 2. Review waves → triage (highest priority)

A teammate's review (e.g. `🟡 Needs triage` items) blocks the merge until it is answered.
For each untriaged finding: check the claim against the code, then sort it as in
`iot-ticket-closeout` §3:

- **Fix** on the PR branch: spec first, then the fix, then `vp check` / `vp test`, then
  push. Restack children afterwards (step 3).
- **Carry** to a downstream ticket: add a `Carried from` note in its body and a comment.
- **Answer / dismiss**: draft the thread reply. Post it only with approval.

**Writing replies, PR notes and carry notes:** never cite the design doc, a ticket or a
PR by its ID alone (`R5`, `P2.1`, `#448`, `MFB-1602`). Say what it is or what it says,
then put the ID in brackets: "the error-handling PR (#450)", "errors carry status and
body ≤ 2,048 chars (R5)". If the description would run long, shorten it to a few words.
Readers often don't have the design doc open.

Then update the PR body (Test plan / Intentionally deferred) so it matches the branch.
One PR per iteration; finish it before moving on.

## 3. Merges → closeout + restack

When a PR has merged: run `iot-ticket-closeout` (Verified comment, tick boxes, carry
findings), then `pr-stack-planner` for the restack commands. Retarget children to `main`
(`gh pr edit <n> --base main`), rebase `origin/<child>` (not a local branch), and push with
a pinned lease, `--force-with-lease=<branch>:<old head SHA>`. A bare lease to the HTTPS URL
is always rejected; never fall back to `--force`. A force push after review needs approval,
so batch these and ask. The handoff doc's restack section has the full recipe and the #450
exception.

## 4. Ticket accuracy

For tickets that are about to start (blockers merged or in review), run `ticket-critic`.
Fix what's fixable now: inline the R-rules it cites (§3.1), add the Slice to the footer,
and add missing links. Questions that need a human go on the ticket under **Open
question** and into `state.md`.

## 5. Start next work (WIP cap)

Only if: open DPS PRs with untriaged reviews = 0, **and** unreviewed DPS PRs < 3.
(Review bandwidth is the bottleneck. More PRs means more drift and more restacks.)
Pick the highest-priority ready ticket: lowest slice first, then the most downstream
unblocked work. Run `iot-work-ticket <KEY>`, and stop before pushing a new PR to ask.

## 6. Write state and pace

Rewrite `state.md` with these sections, in this order: **Epic**, **Watermark**, **In
flight** (empty if idle), **Open PRs** table (number, ticket, base, CI, review state,
untriaged count), **Ready queue**, **Human asks**, **Decisions** (answers the user gave,
so they're never asked again), and **Done**. Keep only the last 3 runs in Done; older runs
are in Jira and git.

Keep the handoff current: if this pass changed a PR, ticket, decision or open item, update
`docs/plans/dps/handoff.md` and the `claude-kit/` copy (`rsync -a --exclude drafts` this
skill's folder into `docs/plans/dps/claude-kit/skills/iot-epic-loop/`), run
`vp fmt docs/plans/dps/`, commit `docs(dps): handoff — <what changed>` and push. Where to push: the handoff PR's
branch while that PR is open; after it merges, the DPS PR branch you are pushing to anyway,
or else the one docs-only branch `docs/mfb-1587-dps-handoff-update` (reuse its PR if one is
open). Opening a new PR for it follows the standing-authority rules above. Never
leave the doc contradicting the branches or Jira.

Then `ScheduleWakeup`: ~1800 s when waiting on reviews; 1200 s if you pushed fixes and CI
is running; stop when the epic has no open work or context is low.
