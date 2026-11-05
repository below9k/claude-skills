---
name: ticket-write
description: Draft, rewrite, or critique Jira tickets against the shared ticket standard (plus the repo profile's ticket specifics). Use when asked to write a ticket, break an epic or design doc into tickets, tighten or rewrite existing tickets, or check whether a ticket is ready to implement. Works on one ticket or a whole epic; drafts locally and iterates with the user; never writes to Jira without approval (hand-off to backlog-sync).
effort: high
---

# Write tickets

Request:

$ARGUMENTS

## Read first

1. `~/.claude/skills/ticket-write/ticket-standard.md` — the contract. Every draft must
   satisfy it. Note which **[open]** rules you applied.
2. The repo profile (`~/.claude/repos/README.md`): its **Tracker** section and `tickets.md`
   (summary and footer format, design-doc location, labels, legacy repo for rewrites).
3. `~/.claude/skills/ticket-write/examples/` — before/after rewrites. Match their level of
   detail.
4. In the repo: its glossary (e.g. `CONTEXT.md`), ADRs, label/triage docs, and
   `AGENTS.md`/`CLAUDE.md`, when present.

## Modes

Work out the mode from the request. Ask only if it's genuinely ambiguous.

| Mode | Input | Output |
| --- | --- | --- |
| **critique** | ticket key(s) or a JQL/epic key | findings per ticket, no rewrite |
| **rewrite** | existing ticket key(s) | proposed new body per ticket + findings it fixes |
| **draft** | an ask, a bug, or a design-doc section | new ticket(s) |
| **breakdown** | a design doc / plan / epic | backlog index (ID, summary, slice, type, size, blocked-by) first, then bodies |

## Workflow

### 1. Gather
- Jira: read with `responseContentFormat: html` and `view: evidence`. Read **comments**
  too; carried-forward findings live there (standard §3.7). For an epic:
  `parent = <KEY> ORDER BY key`.
- Design doc: try Confluence (the profile names the site). If it isn't granted to the MCP,
  look for a local export and say which version you used.
- Open PRs/MRs that touch the area (search by ticket key). Their "Deferred to" and
  "Carried to" lines are requirements for other tickets.

### 2. Ground the claims (in parallel)
- **`legacy-source-scout`** for rewrite tickets: give it the legacy repo (a local clone, or
  the forge API; the profile names it), the component, and every number, state or name the
  draft asserts. It returns verified values with `file:line`, and behaviours the doc omits.
- Check that each `Pattern:` path exists in the repo (a plain `ls` is fine).

### 3. Draft
- Write drafts to a local file first: `$CLAUDE_JOB_DIR/tmp/tickets/<ID-or-KEY>.md` when
  that is set, otherwise a scratch directory. Use one file per ticket, markdown for
  reading. Convert to HTML only at publish time.
- Follow the anatomy in standard §2 exactly. Quote rule rows inline (§3.1). Tag
  `(parity)/(new)/(changed: …)` on rewrite work (§3.3).
- In **breakdown**, show the backlog index and slice table first, and agree it before
  writing bodies. Granularity is the cheapest thing to fix early.

### 4. Critique your own draft
Run the **`ticket-critic`** agent on the draft files, alongside the scout if that hasn't
finished. Fix the blockers it raises. Keep its `question` findings for the user. Don't
silently resolve them.

### 5. Iterate with the user
Present compactly:
- Per ticket: the summary line, what changed vs the current Jira body (in rewrite mode),
  and the critic's open questions.
- Questions that apply to many tickets go once, at the top. Example: "Quote R3 inline in
  all planner tickets, or link a committed reference file?"
- Show at most ~5 full bodies per turn. Offer the rest as files.

When the user corrects a *pattern* rather than one ticket ("never put timeouts in AC",
"always say which feature file"), propose the exact edit to `ticket-standard.md` and its
changelog line, and apply it once they agree. If the rule only holds for one repo or
tracker, put it in that profile's `tickets.md` instead. The standard should get better with every
round, so the same correction isn't needed twice.

### 6. Hand off
Once the user approves the drafts, hand off to **`backlog-sync`** to publish (create or
update, links, labels). Don't write to Jira from this skill. Jira mutations need explicit
approval in the conversation.

## Output

End with:
- the files written (paths)
- tickets ready to publish vs still open, and why
- proposed standard changes (if any) and whether they were applied
