---
name: iot-ticket-closeout
description: Close the loop between a PR and its MFB Jira ticket in FarmBot's `iot` repo — verify each acceptance criterion against the PR, post the standard "Verified in the PR" comment, tick the verified boxes, and carry review findings forward into the downstream tickets they affect. Use after opening a PR for a ticket, after a review wave lands on a PR (e.g. a teammate's triage-status review), or when asked to update Jira from a PR.
effort: medium
---

# Close out an iot ticket

Ticket and/or PR:

$ARGUMENTS

The format comes from §6 of `~/.claude/skills/iot-ticket-write/ticket-standard.md`.
Carry-forward rules come from §3.7. Every Jira write needs explicit approval in this
conversation; show the exact text first.

## 1. Gather

- Ticket: body as HTML, comments, links, parent epic, and siblings (`parent = <epic>`).
- PR: `gh pr view <n> -R farmbot-monitoring-solutions/iot --json
  body,baseRefName,headRefName,files,reviews,comments`, inline review comments
  (`gh api repos/farmbot-monitoring-solutions/iot/pulls/<n>/comments`), and CI status.
- Stack: the base chain (`Stacked on #446 → #444`).

## 2. Verify each criterion

For each AC item, find the proof in the diff: a test, a scenario, a command output. Mark
it:

- `[x]` verified — cite the evidence.
- `[x]` verified, with a correction — "Ticket said 30; the M3S suite has 38. All 38
  ported."
- `[ ]` not met — say what's missing. Don't tick it.

Also list:
- **Kept for parity beyond the ticket:** legacy behaviour the PR preserves that the ticket
  didn't spell out.
- **Deferred:** item → owning key.

## 3. Triage review findings

For each review finding (Codex results in the PR body, and human or agent review comments
such as a teammate's `🟡 Needs triage` items), decide:

| Outcome | Action |
| --- | --- |
| Fixed in this PR | note it in the comment |
| Belongs to a sibling ticket not started yet | **carry**: amend that ticket |
| New work no ticket owns | propose a new ticket via `iot-ticket-write` |
| Dismissed | one-line reason in the PR thread (only if the user wants to reply) |

**Carry** means, for each target ticket: add a `**Carried from <KEY> (PR #n):**` note
under **Notes** in its body (and change the AC if the finding changes "done"), **and**
post a comment so watchers see it. If the finding forces a decision, write out the options
and a recommendation, as in the MFB-1607 duplicate-create note.

## 4. Propose, then write

Show the user, in one message:
1. The closeout comment for this ticket.
2. The checkbox changes (which items flip to `checked`).
3. Each carry-forward edit and comment, per target ticket.
4. Any status transition (e.g. → PR REVIEW). Only when asked, or when it's the obvious
   next state.

After approval: edit the body with HTML from the latest `html` read (to preserve task-list
ids and checked state), then add the comments. Report what was written, with links.

## 5. Feed the standard

If the closeout found a ticket-writing failure (a wrong number, a missing branch, an
unstated rule), end with a one-line proposal for `ticket-standard.md` or the epic's other
tickets ("P3.5 also cites R3 bare; fix before starting").
