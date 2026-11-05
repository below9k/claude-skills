---
name: epic-loop
description: One iteration of the living workflow for a Jira epic delivered as a series of PRs/MRs (often stacked) — syncs PR and Jira state, triages new review waves, closes out and restacks after merges, keeps tickets accurate, and starts the next ready ticket within a WIP cap. Keeps per-epic state in ~/.claude/state/epic-loop/<EPIC>/state.md. Designed to run under `/loop /epic-loop <EPIC>`; stops cleanly when context runs low.
effort: high
---

# Epic loop (one iteration)

Epic: $ARGUMENTS (if empty, use the only directory under `~/.claude/state/epic-loop/`, or ask)

- **State:** `~/.claude/state/epic-loop/<EPIC>/state.md` (outside git; drafts and audits for
  the epic go in the same directory). Read it first and rewrite it last. If it doesn't
  exist, create it with the sections in step 6 and ask the user for the standing authority.
- **Contract:** `~/.claude/skills/ticket-write/ticket-standard.md`.
- **Repo profile:** load it from the repo checkout named in `state.md`
  (`~/.claude/repos/README.md`). It gives the forge, the branch pattern for the epic's PRs,
  push mechanics, the validation gates, and `tickets.md`.
- **Standing authority** is recorded per epic in `state.md` (who granted what, and when).
  Without one, the default is: ask first for every Jira write, every PR/MR comment or reply,
  every push, opening a PR, merging, force-pushing a reviewed branch, and closing or
  transitioning a ticket to Done. Anything outside the epic always needs asking.
- **Epic-specific chores** (keeping a handoff doc in the repo current, and similar) are
  listed in `state.md` under **Chores**; do them in step 6.

## Starting, stopping and resuming

The user starts and stops this loop by hand, often because of token usage: a session runs
out, `/loop` gets cancelled, or a new session starts days later. **Nothing about one run or
one epic lives in this file.** Everything that changes between runs goes in `state.md`, so
this skill never needs editing on a rerun. If you notice yourself wanting to edit SKILL.md to record
progress, a decision or a new PR, put it in `state.md` instead. Edit this file only when
the *process* changes, and say so to the user.

**Starting** (`/loop /epic-loop <EPIC>` or `/epic-loop <EPIC>`, possibly in a fresh session
with no memory of earlier runs):

1. Read `state.md`. If the epic argument is missing, use `state.md`'s epic.
2. Check the **In flight** section. If it names work, finish that before anything else.
3. Recover from a hard stop, because the last run may have died mid-step without writing
   state. Check for:
   - uncommitted changes, or local commits not on the remote
     (`git -C <checkout> status -sb` and `git log @{u}..` on each of the epic's branches);
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
   resume (normally `/loop /epic-loop <EPIC>`).

**Being stopped without warning:** this is why rule 4 matters. Keep each unit of work
small enough that losing it costs one step, and update **In flight** *before* starting a
long step (a ticket implementation or a review-fix wave), not only after.

## 0. Budget guard

Check context at the start of each step. Past ~70%, don't start a new step: do "Stopping on
purpose". Never start a ticket you can't finish in the budget left; record it as next in
**In flight** instead.

## 1. Sync (cheap, every iteration)

- List the epic's PRs/MRs by branch pattern (the profile's `tickets.md` or `state.md` gives
  it). GitHub:
  `gh pr list --search "head:<pattern>" --state all --json number,title,state,baseRefName,headRefName,reviewDecision,statusCheckRollup,updatedAt`.
  GitLab: `list_merge_requests` filtered by source branch.
- For each open PR: CI status, `reviews` and inline comments **newer than `state.md`'s
  watermark**, and commits behind its base.
- Jira: `parent = <epic>` with status, labels and links (compact view; no descriptions
  unless needed).
- Assignee check (every iteration): `parent = <epic> AND status != BACKLOG AND status !=
  Done AND assignee is EMPTY` (status names from the profile). Assign each hit to the
  epic's owner: the current Atlassian user (`atlassianUserInfo` gives the accountId).
  Whenever the loop moves a ticket out of BACKLOG itself, assign it in the same step.
- Diff against `state.md`. If nothing changed, skip to step 5.

## 2. Review waves → triage (highest priority)

A teammate's review (e.g. `🟡 Needs triage` items) blocks the merge until it is answered.
For each untriaged finding: check the claim against the code, then sort it as in
`ticket-closeout` §3 (and `review-follow-up` for the fix-the-class discipline):

- **Fix** on the PR branch: spec first, then the fix, then the profile's gates, then push.
  Restack children afterwards (step 3).
- **Carry** to a downstream ticket: add a `Carried from` note in its body and a comment.
- **Answer / dismiss**: draft the thread reply. Post it only with approval.

**Writing replies, PR notes and carry notes:** never cite the design doc, a ticket or a
PR by its ID alone (`R5`, `P2.1`, `#448`, `PRJ-1602`). Say what it is or what it says,
then put the ID in brackets: "the error-handling PR (#450)", "errors carry status and
body ≤ 2,048 chars (R5)". If the description would run long, shorten it to a few words.
Readers often don't have the design doc open.

Then update the PR body (Test plan / Intentionally deferred) so it matches the branch.
One PR per iteration; finish it before moving on.

## 3. Merges → closeout + restack

When a PR has merged: run `ticket-closeout` (Verified comment, tick boxes, carry findings),
then, if the epic stacks PRs, `pr-stack-planner` for the restack commands. Retarget children
to the merged PR's base, rebase `origin/<child>` (not a local branch), and push with a
pinned lease, `--force-with-lease=<branch>:<old head SHA>`. Never fall back to `--force`.
A force push after review needs approval, so batch these and ask. `pr-stack-planner` has the
full recipe, including the merge-commit exception.

## 4. Ticket accuracy

For tickets that are about to start (blockers merged or in review), run `ticket-critic`.
Fix what's fixable now: inline the design-doc rules it cites (§3.1), complete the footer,
and add missing links. Questions that need a human go on the ticket under **Open
question** and into `state.md`.

## 5. Start next work (WIP cap)

Only if: the epic's open PRs with untriaged reviews = 0, **and** unreviewed PRs < the WIP
cap in `state.md` (default 3).
(Review bandwidth is the bottleneck. More PRs means more drift and more restacks.)
Pick the highest-priority ready ticket: lowest slice first, then the most downstream
unblocked work. Run `work-ticket <KEY>`, and stop before pushing a new PR to ask.

## 6. Write state and pace

Rewrite `state.md` with these sections, in this order: **Epic**, **Watermark**, **In
flight** (empty if idle), **Standing authority**, **Chores**, **Owed and notes**, **Env
notes**, **Open PRs** table (number, ticket, base, CI, review state, untriaged count),
**Ready queue**, **Human asks**, **Decisions** (answers the user gave, so they're never asked
again), and **Done**. Keep only the last 3 runs in Done; older runs are in Jira and git.
Durable repo facts you learn (a gate, a push quirk) belong in the repo profile, not in
`state.md`; propose the edit.

Then do the **Chores** listed in `state.md` (for example, keeping a handoff doc in the
repo in step with the PRs and Jira), following its instructions and the standing authority.
Never leave such a doc contradicting the branches or Jira.

Then `ScheduleWakeup`: ~1800 s when waiting on reviews; 1200 s if you pushed fixes and CI
is running; stop when the epic has no open work or context is low.
