---
name: iot-backlog-sync
description: Publish approved ticket drafts to Jira (MFB project on farmbot.atlassian.net) for FarmBot's `iot` work, and keep a design doc's backlog and Jira in step — create or update issues under an epic, convert "Depends on" into real Jira blocking links, replace P-IDs with keys, apply triage labels, and report drift between the doc and Jira. Use after iot-ticket-write drafts are approved, or when asked to sync, publish, link, or drift-check an epic's tickets.
effort: medium
---

# Sync an iot backlog with Jira

Request:

$ARGUMENTS

Contract: `~/.claude/skills/iot-ticket-write/ticket-standard.md` (§3.6 links, §3.9
labels, §4 source of truth, §5 formatting). **Every Jira mutation needs explicit approval
in this conversation**. Plan first, show the plan, then write.

Jira: cloud `farmbot.atlassian.net` (call `getAccessibleAtlassianResources` once for the
cloudId), project `MFB`. Design docs live on `farmbot-au.atlassian.net`. If that site
isn't granted to the MCP, use a local export (e.g. `~/Documents/Epic- *.md`) and say so.

## Modes

### publish
Input: approved draft files from `iot-ticket-write`, plus the epic key.

1. Match each draft to an existing issue: by key if the draft has one, otherwise by the
   `P#.#` prefix in summaries under the epic. Unmatched drafts are **creates**.
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
   - Replace `P#.#` references in bodies and footers with keys.
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
