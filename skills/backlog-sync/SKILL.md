---
name: backlog-sync
description: Publish approved ticket drafts to Jira, and keep a design doc's backlog and Jira in step — create or update issues under an epic, convert "Depends on" into real Jira blocking links, replace design-doc IDs with keys, apply triage labels, and report drift between the doc and Jira. Use after ticket-write drafts are approved, or when asked to sync, publish, link, or drift-check an epic's tickets.
effort: medium
---

# Sync a backlog with Jira

Request:

$ARGUMENTS

Contract: `~/.claude/skills/ticket-write/ticket-standard.md` (§3.6 links, §3.9
labels, §4 source of truth, §5 formatting). **Every Jira mutation needs explicit approval
in this conversation**. Plan first, show the plan, then write.

Jira site, project and design-doc location: from the repo profile's **Tracker** section and
`tickets.md` (`~/.claude/repos/README.md`); otherwise ask. Call
`getAccessibleAtlassianResources` once for the cloudId. If the design-doc site isn't granted
to the MCP, use a local export and say so.

## Modes

### publish
Input: approved draft files from `ticket-write`, plus the epic key.

1. Match each draft to an existing issue: by key if the draft has one, otherwise by the
   design-doc ID prefix in summaries under the epic (e.g. `P#.#`; see the profile). Unmatched drafts are **creates**.
2. Build the plan table: `ID | key or NEW | action (create/update/none) | fields changed |
   links to add | labels`. For updates, show a short diff of the description (sections
   added, removed or changed). Don't paste whole bodies.
3. After approval:
   - Create in **dependency order**, so blockers get keys first.
   - Write descriptions as HTML (task lists per §5). For updates, start from the latest
     `responseContentFormat: html` read so checked boxes survive. Never untick a box.
   - Set `parent` to the epic.
   - Add **"Blocks" links** for every `Blocked by` entry (find the link operation through
     `discover` if it isn't a primary tool). Check existing links first; don't duplicate.
   - Replace design-doc ID references in bodies and footers with keys.
   - Apply labels only if the user agreed the §3.9 lifecycle.
4. Report: created and updated keys with links, links added, and anything skipped (with
   the reason).

### drift
Input: epic key and design doc (Confluence or local export).

Compare the doc's backlog and reference sections with Jira, and report:
- Tickets in one place but not the other.
- Summary, size or dependency differences.
- AC differences per ticket (added, removed or changed items).
- Terms renamed in one place only (e.g. `T_M3S` vs `T_DPS`, "D1 to D9" vs "D1 to D11").
- Jira tickets with `Depends on` text but no links.

For each difference, recommend which side to change, using §4 (doc owns reference
material, Jira owns ticket bodies), and offer the edits. Confluence edits go through
`updateConfluenceContent` only if that site is granted. Otherwise, output a patch the user
can paste.

### links
Only the §3.6 repair: read every child of the epic, parse `Depends on:` / `Blocked by:`,
and propose the missing "Blocks" links as one batch for approval.
