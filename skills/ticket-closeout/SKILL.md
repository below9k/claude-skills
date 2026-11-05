---
name: ticket-closeout
description: Close the loop between an implementation (a PR/MR on GitLab or GitHub, or a branch) and its Jira ticket — verify each acceptance criterion against the change, post the standard "Verified in the PR" comment, tick the verified boxes, and carry review findings forward into the downstream tickets they affect. Use when implementation is done (work-ticket hands off here), after opening a PR/MR, after a review wave lands, or when asked to update Jira from a PR/MR.
effort: medium
---

# Close out a ticket

Ticket and/or PR/MR:

$ARGUMENTS

The comment shape comes from §6 of `~/.claude/skills/ticket-write/ticket-standard.md`, and
the carry-forward rules from §3.7. Load the repo profile (`~/.claude/repos/README.md`) for
the tracker, forge and any `tickets.md` specifics. **Every Jira write needs explicit
approval in this conversation** unless a standing authority covers it; show the exact text
first.

## 1. Gather

- Ticket: body (read as HTML, `responseContentFormat: html`, so checked state and ids
  survive), comments, links, parent epic, and siblings (`parent = <epic>`).
- Change:
  - GitHub: `gh pr view <n> --json body,baseRefName,headRefName,files,reviews,comments`,
    inline comments (`gh api repos/<o>/<r>/pulls/<n>/comments`), and checks.
  - GitLab: the MR, its diff, discussions, and pipeline (GitLab MCP).
  - No PR/MR yet: the branch diff against its base, and the validation you ran.
- Stack, if any: the base chain (`Stacked on #446 → #444`).

## 2. Verify each criterion

For each AC item, find the proof in the change: a test, a scenario, a command output, a
runtime check. Mark it:

- `[x]` verified: cite the evidence.
- `[x]` verified, with a correction: "Ticket said 30; the legacy suite has 38. All 38
  ported."
- `[ ]` not met: say what's missing. Don't tick it.

Also list:
- **Kept for parity beyond the ticket**: legacy behaviour the change preserves that the
  ticket didn't spell out.
- **Deferred**: item → owning key.

## 3. Triage review findings

For each finding (adversarial-review/Codex results in the PR body, human or agent review
comments such as a teammate's `🟡 Needs triage` items), decide:

| Outcome | Action |
| --- | --- |
| Fixed in this change | note it in the comment |
| Belongs to a sibling ticket not started yet | **carry**: amend that ticket |
| New work no ticket owns | propose a new ticket via `ticket-write` |
| Dismissed | one-line reason in the thread (only if the user wants to reply) |

**Carry** means, for each target ticket: add a `**Carried from <KEY> (<PR/MR>):**` note under
**Notes** in its body (and change the AC if the finding changes "done"), **and** post a
comment so watchers see it. If the finding forces a decision, write out the options and a
recommendation.

## 4. Propose, then write

Show the user, in one message:
1. The closeout comment (standard §6), including a short **Changes** list and the
   **Verification** commands with results.
2. The checkbox changes (which items flip to checked).
3. Each carry-forward edit and comment, per target ticket.
4. Any status transition, only when asked or when it's the obvious next state.

After approval: edit the body with HTML built from the latest HTML read (to keep task-list
ids and checked state), then add the comments. Report what was written, with links.

Never mark a ticket QA-approved: implementation verification and QA are separate.

## 5. Feed the standard

If the closeout found a ticket-writing failure (a wrong number, a missing branch, an
unstated rule), end with a one-line proposal for `ticket-standard.md`, the profile's
`tickets.md`, or the epic's other tickets ("P3.5 also cites R3 bare; fix before starting").
