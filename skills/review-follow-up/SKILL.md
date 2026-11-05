---
name: review-follow-up
description: Address code review feedback on an existing merge request or pull request (GitLab or GitHub), or on a local review report, from start to finish. Use when asked to follow up on, address, respond to, or fix review comments, MR/PR threads, or reviewer feedback, e.g. "address the comments on !123", "triage the review on #449", or "follow up on PRJ-1234's review". Gathers every unresolved thread, verifies each claim against the code, triages it (fix, already handled, answer, push back, carry to another ticket, out of scope), fixes the class of defect rather than only the flagged line, re-validates along the ticket's tracer bullet, pushes new commits to the same branch, and drafts thread replies for the user to approve.
effort: high
---

# Review Follow-up

Follow up on the review feedback identified in:

$ARGUMENTS

This is the second half of `work-ticket`. Reuse its steps by reference rather than
re-deriving them: the tracer bullet (step 5), testing (step 7), the iteration loop (step 9),
runtime and browser verification (steps 10–11), and the final validation and specialist
reviewers (step 13). Load the repo profile first (`~/.claude/repos/README.md`): it names the
forge, remotes, validation gates and agents. Repository `CLAUDE.md`/`AGENTS.md` instructions
override this skill where they conflict.

Forge operations below are given for both: GitLab MCP (`mr_discussions`,
`create_merge_request_discussion_note`, `update_merge_request`) and GitHub `gh`
(`gh pr view <n> --json reviews,comments`, `gh api repos/<o>/<r>/pulls/<n>/comments`,
`gh api …/pulls/<n>/comments/<id>/replies -f body=…`, `gh pr edit <n> --add-reviewer`).

## Objective

Every piece of feedback ends in exactly one of these states, with evidence:

* **fixed**: changed, validated, and the commit named in the reply
* **already handled**: the code already does it, and the reply says where
* **answered**: a question, answered from the code
* **pushed back**: the reviewer's claim is wrong or the change would be worse, with evidence
* **carried**: valid, and it changes how a *later* ticket must be built; it goes into that
  ticket via `ticket-closeout`'s carry-forward
* **deferred**: valid but out of scope, with a follow-up the user approved
* **needs decision**: the user has to choose

Never leave a thread silently unaddressed, and never "fix" something you have not confirmed
is actually wrong.

## 1. Identify the MR and its state

Resolve the input to an MR:

* MR number or URL: use it.
* Ticket key: find MRs/PRs whose source branch contains the key (`list_merge_requests`, or
  `gh pr list --search <key>`).
* Nothing given: use the current branch's MR.
* A local review report (for example `adversarial-review` output or pasted findings): treat
  each finding as a thread with no MR to reply to, and skip step 8.

Record the MR's source branch, target (release) branch, head SHA, pipeline status, and the
Jira ticket.

Make the local checkout match the MR head before changing anything:

* fetch the fork remote and check that the local branch is not behind or diverged from the
  MR's source branch
* do not discard uncommitted work; if the tree is dirty with unrelated changes, stop and say so
* when the branch has diverged, inspect the divergence before doing anything else (see
  `push`)

## 2. Gather the feedback

Read every discussion on the MR/PR, including inline review comments and review summaries.
Keep the ones that are:

* resolvable and unresolved
* unresolved non-diff notes that ask for something
* resolved threads whose last word is the reviewer's, which can hide an unanswered question

Skip system notes and threads the user already answered where the reviewer has not replied
since.

For each thread capture: the reviewer, the file and line (and whether that position is now
outdated), the full thread text including earlier replies, and what is being asked.

Also treat a failing pipeline or check as feedback: read the failing job's log and add it
to the list.

## 3. Rebuild the context

Read the Jira ticket and its acceptance criteria. Then recover the tracer bullet for the
change, from the `work-ticket` completion summary or Jira "Implementation Complete" comment
when it exists, or by reconstructing it from the diff against the target branch
(`work-ticket` step 5). Keep it short. It is needed to judge each comment's blast radius.

## 4. Verify and triage each item

For every item, read the code it points at, and the code around it, before deciding. The
reviewer may be wrong, may be looking at an outdated revision, or may have spotted a symptom
whose cause is elsewhere.

Classify each item into one of the objective's states. For each:

* restate the concern in one line
* give the evidence (file:line, a spec, a log, a reproduction)
* place it on the tracer bullet: which step or boundary it concerns
* for a fix, sketch the intended change

Present the triage table to the user. Continue straight into the fixes for items that are
unambiguously **fixed** work. Wait for the user only on **pushed back**, **deferred**, and
**needs decision** items, and on any fix that changes behavior beyond what the reviewer
asked for.

## 5. Fix the class, not the instance

For each confirmed defect, before fixing it:

1. Name the defect class in one line (for example "subscription not removed when the region
   unmounts", or "error from the device command swallowed before reaching the caller").
2. Search the rest of the MR's diff, and every boundary on the tracer bullet, for the same
   class. Reviewers usually flag one instance, so fix every instance the change introduced.
3. Note any pre-existing instances outside the change separately. Do not fix them unless the
   user asks.

Then fix it, following `work-ticket` step 6: the smallest change that addresses the concern,
in existing patterns, with no unrelated cleanup. When a fix changes the tracer bullet, update
the tracer bullet.

For a confirmed bug, add a regression test that fails before the fix (`spec-author` can
write it). Do not weaken existing tests to accommodate a review change.

## 6. Re-validate

* Run the narrow validation for each fix as you go (the profile's gates, or `spec-runner`).
* Re-run the `work-ticket` step 13 tracer bullet self-review on every step the fixes touched,
  plus their neighbors on the path.
* For a non-trivial follow-up diff, run the specialist reviewers on the full MR diff, not
  just the follow-up commits (`conventions-reviewer` and `async-bug-hunter` in parallel).
  Fix anything new they confirm before going further.
* Do runtime or browser verification when a fix changes observable behavior.

## 7. Commit and push

Use the `push` skill to commit and push to the existing branch. Do not open a new MR/PR.

* Add new commits on top. Do not amend, squash, rebase, or force-push a branch under review,
  because reviewers rely on "changes since last review". Do it only if the user asks.
* Group commits by concern, one coherent commit per piece of feedback or defect class, in
  the repository's commit style.
* Record each commit's short SHA against the threads it addresses.

## 8. Reply on the threads

Draft one reply per thread and show all the drafts to the user before posting anything. Every
post appears as the user.

* **fixed**: what changed, and the commit SHA. Mention other instances of the same class
  that were also fixed.
* **already handled**: where and how, with file:line.
* **answered**: the answer, grounded in the code.
* **pushed back**: the reasoning and evidence, written as a colleague would. No
  defensiveness, no over-apology.
* **deferred**: the follow-up ticket key, or that the item is deferred and why.

Keep replies short and in the user's voice. Do not say a reply was written by an AI unless
the user wants that.

After approval, post them as replies inside the existing threads, not as new top-level
comments (the profile notes forge-specific quirks, such as token scopes).

Resolve threads only when the user says to. By default, leave resolution to the reviewer, and
ask once whether the user wants the **fixed** and **already handled** threads resolved.

If the user wants it, re-request review from the reviewers.

For **carried** items, hand off to `ticket-closeout` (its carry-forward step) once the
user agrees, so the downstream ticket's body and watchers both get the finding.

## 9. Feed lessons back

The point of this loop is to catch more of this feedback before review next time. For each
confirmed defect class, ask whether `work-ticket`'s tracer bullet steps (5, 6, 7, and the
13 self-review) or a repository `CLAUDE.md` would have caught it. If a class recurs, or
reflects a non-obvious codebase rule, propose a one-line addition to the self-review
questions or to the relevant `CLAUDE.md`. Propose it; do not make the edit without the
user's agreement.

## 10. Report

Report:

* MR, branch, and the new commits
* the triage table with each item's final state
* defect classes found, and other instances fixed
* validation run, and results
* replies posted (or drafted and awaiting approval) and threads resolved
* anything deferred or awaiting a decision
* proposed `work-ticket` / `CLAUDE.md` lessons

Do not transition the Jira ticket. Do not mark it QA-approved.

## Completion standard

The follow-up is complete when every unresolved thread and failing pipeline job is in one of
the objective's states with evidence, the fixes are validated and pushed to the MR branch,
the tracer bullet self-review holds for the touched steps, and the replies have been posted
or handed to the user as drafts.
