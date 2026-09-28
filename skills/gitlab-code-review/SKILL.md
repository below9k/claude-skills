---
name: gitlab-code-review
description: >
  Use this skill whenever the user asks for a code review, review of
  changes, review of a commit, review of a branch, review before creating
  or updating a merge request, or assessment of changes for correctness,
  regressions, maintainability, or quality. When a Jira ticket can be
  identified, inspect the Jira issue and acceptance criteria and verify
  that the implementation satisfies the stated requirements. For GitLab repositories,
  determine the intended release/base branch before reviewing. GitLab
  feature branches typically originate from release/v#.#.# or
  release/prj/v#.#.# branches, and reviews should compare the feature
  branch against that base branch rather than automatically comparing
  against main, master, or develop. Review the actual diff and relevant
  surrounding code, not just the changed lines. Every review also runs
  an iterative adversarial review through Codex CLI, whose findings are
  verified and merged into the final review. GitLab repositories are
  located in folders named like ~/develop/savi*.
---

# Code Review

## Purpose

Perform a thorough, practical code review focused on finding real problems
rather than producing stylistic commentary.

Prioritize:

1. Bugs and incorrect behavior.
2. Regressions.
3. Security issues.
4. Incorrect assumptions.
5. Error handling and failure cases.
6. Concurrency and race conditions.
7. Resource leaks.
8. API/interface compatibility.
9. Performance problems.
10. Maintainability problems that are likely to cause future defects.

Do not report purely stylistic issues unless they conflict with established
repository conventions or materially affect maintainability.

---

## Determine Review Scope

First determine what should be reviewed.

Possible review targets include:

- Current working-tree changes.
- A specific commit.
- A series of commits.
- The current feature branch.
- A merge request.
- Changes between the current branch and its base branch.

When reviewing a feature branch, review the complete diff from the appropriate
base branch.

Do not assume that the most recent commit represents the entire change.

---

## Jira Requirements

When the change is associated with a Jira ticket, inspect the corresponding
Jira issue before completing the code review.

You can find the Jira ticket reference in the branch name, commit messages,
merge request title or description, or within the changed code itself.
Tickets usually look like SN-####, DSP-###, GAL-####, or similar.

Use the Jira issue to establish the intended behavior and acceptance criteria,
rather than relying solely on the implementation or the user's description.

Determine the Jira ticket from:

* The user's request.
* The feature branch name.
* Commit messages.
* Merge request title or description.
* References in the changed code.

If a Jira ticket can be identified, retrieve the issue and review:

* Summary.
* Description.
* Acceptance criteria.
* Requirements.
* Linked issues when relevant.
* Comments containing clarification or implementation requirements.

Compare the implementation against the Jira requirements.

Specifically check for:

* Requirements that are not implemented.
* Acceptance criteria that are only partially satisfied.
* Behavior that contradicts the Jira requirements.
* Edge cases explicitly described in the ticket.
* Required error handling that is missing.
* Required UI, API, device, or integration behavior that is missing.
* Changes that introduce behavior outside the intended scope of the ticket.

Treat Jira requirements as evidence of intended behavior, but do not assume
that every statement in the ticket is still current if the issue contains
later clarification or contradictory information. Prefer the most recent
explicit clarification.

If the Jira issue cannot be accessed, do not invent its requirements.
Continue the code review using the available repository and change context,
and explicitly note that Jira requirements could not be verified.

If the implementation appears to conflict with a Jira requirement, include
the requirement in the finding and explain how the implementation differs.

Example:

```
[HIGH] Acceptance criterion is not implemented

File: src/example.ts:42

Jira PRJ-567 requires the device to retry the connection after a
disconnect. The current implementation closes the connection without
scheduling a retry.

This means the implementation does not satisfy the acceptance criterion.
```

Do not report a Jira requirement as a code defect merely because the ticket
contains an ambiguous or non-technical statement. If the requirement itself
is unclear, identify the ambiguity separately rather than inventing the
intended behavior.

---

## GitLab Base Branch

For GitLab repositories, inspect the configured remotes:

    git remote -v

Typically:

    origin    -> user's fork
    upstream  -> canonical repository

Inspect branches:

    git fetch upstream
    git branch -a

Determine the feature branch's intended base branch.

Common release branch patterns:

    release/v#.#.#
    release/PRJ/v#.#.#

Examples:

    release/v3.2.1
    release/PRJ/v3.2.1

The review should normally compare:

    <feature-branch> vs <base-release-branch>

Do not automatically compare against:

    main
    master
    develop

unless repository context establishes that one of those is the correct
base branch.

If the branch itself contains enough context to identify its parent release
branch, use that information.

If multiple release branches are plausible and the correct base cannot be
determined confidently, ask the user before proceeding with a branch-level
review.

---

## Establish the Diff

Once the base branch is known, inspect the actual changes.

Use commands such as:

    git diff <base-branch>...HEAD

and:

    git log --oneline <base-branch>..HEAD

The three-dot diff is important because the review should represent the
changes introduced by the feature branch relative to its base.

Also inspect:

    git status
    git diff

when working-tree changes may be relevant.

Do not limit the review to the output of `git diff` if understanding the
change requires inspecting surrounding code.

---

## Understand Before Judging

For each significant change:

1. Read the changed code.
2. Identify the code it interacts with.
3. Understand the expected behavior.
4. Trace important data/control flow.
5. Check error and failure paths.
6. Check callers and consumers when relevant.
7. Check tests covering the behavior.
8. Determine whether the implementation actually satisfies the intended
   behavior.

Use repository documentation, types, interfaces, tests, and existing
patterns as evidence of intended behavior.

Do not assume a change is incorrect merely because it differs from a
preferred implementation.

---

## Review Categories

### Correctness

Look for:

- Incorrect logic.
- Wrong conditions.
- Off-by-one errors.
- Incorrect state transitions.
- Incorrect assumptions about data.
- Null/undefined handling.
- Incorrect asynchronous behavior.
- Incorrect API usage.
- Incorrect protocol behavior.

### Regression Risk

Determine whether the change can break:

- Existing functionality.
- Existing consumers.
- Backward compatibility.
- Existing device/platform behavior.
- Error handling.
- Upgrade/downgrade paths.

Pay particular attention to changes that alter shared utilities, APIs,
state management, or common components.

### Error Handling

Check:

- Network failures.
- Missing data.
- Invalid input.
- Timeouts.
- Partial failures.
- Unexpected responses.
- Process/device failures.
- Cleanup after failures.

Do not assume errors cannot happen merely because the normal path succeeds.

### Async / Concurrency

Look for:

- Race conditions.
- Duplicate requests.
- Stale state.
- Missing awaits.
- Unhandled promises.
- Event listener leaks.
- Timers that are never cleared.
- Resources used after disposal.

### Resource Management

Check:

- File descriptors.
- Sockets.
- Processes.
- Event listeners.
- Timers.
- Streams.
- Subscriptions.
- Database connections.

Ensure resources are cleaned up appropriately on both success and failure.

### Security

Look for:

- Credential exposure.
- Injection vulnerabilities.
- Unsafe input handling.
- Improper authorization.
- Sensitive information in logs.
- Unsafe filesystem operations.
- Trusting unvalidated external data.

Do not manufacture security concerns without a plausible attack or failure
path.

### Performance

Look for meaningful issues such as:

- Unnecessary repeated work.
- Excessive network requests.
- Unbounded memory growth.
- Expensive operations in hot paths.
- Large unnecessary allocations.
- Blocking operations.

Do not flag minor theoretical optimizations unless they are relevant to the
actual code path.

### Testing

Determine whether the change has adequate tests.

Consider:

- New behavior.
- Changed behavior.
- Error cases.
- Regression cases.
- Boundary conditions.

If a test is clearly warranted but missing, report it.

Do not require tests for trivial changes where they provide little value.

---

## Findings

Only report findings that are actionable and supported by the code.

Each finding should include:

- Severity.
- Location.
- Problem.
- Why it matters.
- Suggested fix when appropriate.

Use these severity levels:

### Critical

Likely to cause severe security issues, data loss, production outages, or
system-wide failure.

### High

Likely to cause significant incorrect behavior, crashes, regressions, or
production failures.

### Medium

A meaningful bug or reliability problem that may affect users or specific
conditions.

### Low

A minor correctness, maintainability, or reliability issue that is still
worth addressing.

Do not inflate severity.

---

## Finding Format

Use this format:

    [HIGH] <short description>

    File: <file>:<line>

    <Explain the problem and the conditions under which it occurs.>

    <Explain the impact.>

    Suggested fix: <fix or direction when useful.>

Keep findings concise and specific.

---

## Avoid False Positives

Before reporting a finding:

1. Verify that the behavior actually occurs.
2. Check whether another part of the code prevents the problem.
3. Check existing validation and error handling.
4. Check whether the behavior is intentional.
5. Check repository conventions.
6. Avoid speculative concerns without evidence.

If something is uncertain, explicitly identify it as uncertain rather than
presenting it as a definite defect.

---

## Adversarial Review (Codex CLI)

Every review includes an adversarial pass by Codex CLI, run as a loop
alongside your own review. Codex is a second, independent reviewer whose
job is to find what you missed and to attack what you concluded. Its
output is evidence to verify, not a verdict to copy.

### Setup

Create a working directory in your scratchpad (or `$TMPDIR` if none) for
the prompts and outputs of each round, e.g. `<scratch>/codex-review/`.

Confirm Codex is available with `codex --version`. If it is missing or
fails to authenticate, continue with your own review and state in the
final summary that the adversarial review could not be run and why.

### Invocation

Always run Codex read-only, non-interactively, in the repository root,
with the prompt on stdin and the final message written to a file:

    codex exec \
      --sandbox read-only \
      --ephemeral \
      -C <repo-root> \
      -o <scratch>/codex-review/round-<n>.md \
      - < <scratch>/codex-review/round-<n>-prompt.md

Use a 10-minute Bash timeout, or run it in the background and continue
your own review while it works. Never use
`--dangerously-bypass-approvals-and-sandbox` or a writable sandbox: the
reviewer must not modify the repository.

Every prompt must give Codex:

- The exact diff command for the review scope (e.g.
  `git diff <base-branch>...HEAD`) and the base/feature branch names, so
  it reviews the same change you are.
- The Jira requirements and acceptance criteria when available.
- The severity levels and Finding Format from this skill, plus, per
  finding, the evidence (code path, inputs, or sequence of events) that
  makes it occur.
- An instruction to report only findings it has traced in the code, and to
  state explicitly when it found nothing.

### Round 1 — Independent attack

Start Codex as early as possible, as soon as the diff and Jira context are
established, so it runs while you do your own review.

Do **not** give Codex your findings in round 1; an independent pass avoids
anchoring both reviewers on the same issues. Instruct it to act as a
skeptical adversarial reviewer trying to prove the change is wrong:
broken edge cases, failure and cleanup paths, races, regressions in
callers and consumers, unmet acceptance criteria, and missing tests.

### Reconcile

When the round finishes, read its output and handle every Codex finding
individually:

1. Open the cited code and verify the claim yourself, applying
   **Avoid False Positives**.
2. Classify it as:
   - **Accepted** — verified; merge it into your findings (deduplicate
     against yours and keep the more accurate severity and explanation).
   - **Rejected** — disproven; record the specific evidence (the guard,
     caller, test, or invariant that prevents it).
   - **Uncertain** — plausible but not provable from the code; keep it
     and label it uncertain.
3. Where Codex found a class of problem you missed, look for the same
   pattern elsewhere in the diff. Your own review continues through
   every round; it is not finished when round 1 starts.

### Round 2+ — Challenge

Run another round with a prompt containing:

- The current merged findings list.
- Each rejected Codex finding with your evidence for rejecting it.
- An instruction to: (a) rebut any rejection it still believes is wrong,
  with new evidence; (b) attack the merged findings for false positives
  or wrong severity; (c) look for defects that neither reviewer has
  reported yet, especially in areas not yet discussed.

Reconcile its response the same way. A rebuttal that brings new evidence
reopens the finding; one that only restates the claim does not.

### Stopping

Stop the loop when a round produces no new accepted findings, no
successful rebuttals, and no severity changes, or after 3 Codex rounds,
whichever comes first. If a disagreement is still open after the last
round, do not silently drop it: report it under **Disputed** with both
positions and the evidence for each.

### Fixes

The review does not modify code unless the user asked for fixes. If they
did, fix the accepted findings after the loop ends, then run one more
round of both your review and Codex on the updated diff to catch
regressions introduced by the fixes.

---

## Final Review Summary

End the review with:

    Base branch: <base branch>
    Reviewed branch: <feature branch>

    Findings:
    - Critical: #
    - High: #
    - Medium: #
    - Low: #

    Adversarial review (Codex): <rounds run, or why it was not run>
    - Codex findings accepted: #
    - Codex findings rejected: #
    - Disputed: #

    Overall assessment: <summary>

Tag each finding with its source: `(Claude)`, `(Codex)`, or `(both)`.
List disputed items after the findings, each with both positions. Briefly
list rejected Codex findings with the one-line reason for rejection, so
the reader can check the reasoning.

The overall assessment should state whether the changes appear ready to
merge and identify any significant remaining concerns.

If there are no findings, explicitly state that no actionable defects were
identified.

Do not claim that code is guaranteed correct merely because no issues were
found.
