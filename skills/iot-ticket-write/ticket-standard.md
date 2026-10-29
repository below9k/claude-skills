# FarmBot `iot` ticket standard

**Status: v0 draft, under discussion.** Derived from the DPS rewrite epic (MFB-1587). Every
rule below has a *Why* tied to something that went wrong or right there. Rules marked
**[open]** are still being debated. Follow the current recommendation, and in the first
reply that uses the rule, say which open rule you applied. When the user settles one,
update this file and record the decision in the changelog at the bottom.

This file is the single contract. `iot-ticket-write`, `iot-backlog-sync`, `iot-work-ticket`,
`iot-ticket-closeout`, `ticket-critic` and `legacy-source-scout` all read it. Change it here,
not in those files.

---

## 1. Who a ticket is for

Read in order of who suffers most from a bad ticket:

1. **The implementer**, often an agent with no memory of the planning conversation. It
   must be able to start from the ticket and the code alone.
2. **The PR reviewer** (e.g. a teammate running their own review wave). They judge "done"
   against the AC, so the AC must be checkable without opening the design doc.
3. **Ops / product** quoting behaviour later. Rules they would quote belong in a
   `.feature` file (ADR 0003), and the ticket says which one.

*Why:* DPS tickets were written for the author, who had the TSD open. Reviewers and future
agents got `see R3`.

## 2. Anatomy

### Story (code that changes behaviour)

```
Summary:  P3.7 Dispatcher loop                       ← "<ID> <outcome noun phrase>"

### Why
<1–2 sentences: what this leaves working, which slice it advances, who notices.>

### Behaviour
<What the system does after this ticket, in domain words (CONTEXT.md glossary).
 Not a file list. Include the rule rows this ticket implements, quoted inline (§3.1).>

### Acceptance criteria                              ← Jira HTML task list (§5)
- [ ] <observable outcome> — <how it is proven: test name / .feature / command>
- [ ] ...

### Out of scope
- <thing a reader might assume is included> → <ticket key that owns it, or "not planned">

### Notes
- **Pattern:** `<path that exists today>` — <what to copy from it>
- **Source (parity):** `<legacy repo path:line>` — <what it pins>
- **Carried from <KEY> (<PR>):** <finding that changes how this ticket is built>

---
ID: P3.7 · Slice: 3 · Size: L · Blocked by: MFB-1597, MFB-1599, MFB-1606 (as Jira links)
Source: <design-doc anchor>
```

### Task (decisions, access, ops, cutover — no behaviour change in code)

```
### Outcome
<The state of the world when this is done.>

### Checklist                                        ← task list; each item has an owner if not the assignee
- [ ] ...

### Evidence of done
<What gets attached or linked: ADR PR, ticket comment with decisions, screenshot, runbook URL.>

---
ID · Slice · Size · Blocked by · Source
```

*Why:* P0.x and P8.x are Tasks, and "Acceptance criteria" read oddly on them. P0.1's last
item ("open questions assigned to owners") was never ticked because nobody owned it.

## 3. Rules

### 3.1 No bare references to the design doc
AC and Behaviour never depend on text the reader can't see. If a criterion rests on doc
rule R3, quote the rows it needs (a small table is fine) or enumerate them. A link to the
doc stays in the footer for context.

- Bad: `All rules in R3, first match wins.`
- Good: the 11 rows of the planner table inline under **Behaviour**, then
  `- [ ] Each row has a scenario in planner.feature; first match wins.`

*Why:* MFB-1604 ("All rules in R3"), MFB-1602 ("Every row in R8") and MFB-1588 ("Review
section 6") are unreadable without the TSD. Decision IDs (D5)
and goal IDs (G3) are the same: name the decision ("`Scheduled` state dropped (D5)").

**[open] 3.1a** Quote inline, or move the reference tables into `docs/` in the repo (for
example `docs/plans/dps/reference.md`) and link to a file at a commit? Recommendation:
quote inline for rules the AC test. Rules that span many tickets (R3, R4) are also worth
committing to the repo so the PR can update them.

### 3.2 Every number and name is sourced
Constants, counts, state names, field names and paths are either verified against the
source (legacy repo, `iot` code or design doc) with a citation in **Notes**, or marked
`(unverified)`. `legacy-source-scout` does the check.

*Why:* MFB-1604 said "30 ported M3S cases"; the suite has 38. The closeout caught it, but
only after the work.

### 3.3 Mark parity vs new vs changed
Rewrite tickets tag each criterion's origin:

- `(parity)` — the legacy system does this today. The legacy code wins over the doc if
  they disagree; raise the gap.
- `(new)` — legacy doesn't do it.
- `(changed: <decision>)` — deliberately different. Name the decision.

*Why:* G3 ("same decisions as M3S") was the goal, but the AC didn't say which behaviour was
parity. The closeout had to explain "kept for parity, beyond the table".

### 3.4 Criteria are observable and proven
Each criterion names an outcome someone can check, and how: a test or scenario name, a
command, or a deploy check. Avoid "works", "handles", "robust" and "supports" with no
observable consequence.

Implementation details (`redirect: "manual"`, `AbortSignal.timeout(55 s)`) are fine **when
they are the requirement**, such as a security or ops policy a reviewer must check. They
are not fine when they only describe how one engineer would do it.

### 3.5 Size and split
- 2–6 criteria. More than 6, or two unrelated verbs in the summary, means split.
- **L** (1–2 weeks) is a smell. Prefer two **M** tickets, each leaving something verifiable.
- **[open] 3.5a** Keep S/M/L or move to points? Recommendation: keep S/M/L in the footer;
  the team doesn't use a points field.

### 3.6 Dependencies are Jira links
`Blocked by` uses real Jira "is blocked by" links, not just footer text. After creation,
replace `P1.3`-style IDs with keys everywhere (body, footer, design doc).

*Why:* every DPS ticket has `issuelinks: []`. The stack order had to be worked out from
text.

### 3.7 Discoveries amend the ticket, not just the comments
When review on ticket A changes how ticket B must be built, B's body gets a
`**Carried from A (PR):**` note under **Notes** (and an AC change if needed), plus a comment
so watchers are notified. Comments alone get lost.

**[open] 3.7a** Edit the body, or comment only? Recommendation: both, as above. The body is
what the next implementer reads.

*Why:* the duplicate-create risk from PR #448 lives only in a comment on MFB-1607.

### 3.8 Out of scope names an owner
Every deferral points to a key (`→ MFB-1599`) or says `not planned`. Mirror it in the PR's
"Intentionally deferred".

### 3.9 Labels and readiness
**[open] 3.9a** Proposed lifecycle, using the repo's triage vocabulary
(`docs/agents/triage-labels.md`):

| Label | Means |
| --- | --- |
| `needs-triage` | Drafted, not reviewed against this standard |
| `needs-info` | `ticket-critic` or a human found a blocking gap |
| `ready-for-agent` | Passes the critic with no blockers; numbers sourced |
| `ready-for-human` | Needs judgement, access or a decision an agent can't make |

DPS tickets all carry `triage`. Changing labels is a Jira mutation, so it needs explicit
approval in the conversation (per `triage-labels.md`).

## 4. The design doc ↔ Jira relationship

**[open] 4a** Which is the source of truth after the tickets exist?

Today the TSD generated the tickets and then both drifted: the doc says "D1 to D9" and
`T_M3S`, Jira says "D1 to D11" and `T_DPS`. Recommendation: the doc owns **reference**
material (§7-style rules, goals, decisions); **Jira owns ticket bodies** once created. The
doc's backlog section shrinks to an index of keys. `iot-backlog-sync --drift` reports
differences either way.

## 5. Jira formatting

- Write descriptions as **HTML** (`contentFormat: html`). Checkboxes must be
  `<ul data-type="task-list"><li data-type="task-item"><input type="checkbox"> …</li></ul>`.
  Markdown `- [ ]` renders as literal text.
- Use `<h3>` for section headings, `<code>` for identifiers, and `<table>` for quoted rule
  rows.
- Read with `responseContentFormat: html` before editing, so checked state and panels
  survive.
- Site: `farmbot.atlassian.net`, project `MFB`. The design docs live on
  `farmbot-au.atlassian.net`, which may not be granted to the MCP; fall back to a local
  export (DPS: `~/Documents/Epic- Rewrite M3S as the Device Provisioning System.md`) and
  say so. Exports drop the collapsed ticket headings, so match bodies by order within a
  phase.

## 6. Closeout comment shape

Posted when the PR is up (by `iot-ticket-closeout`):

```
PR: #449 (stacked on #446 → #444). Verified in the PR:
- [x] <criterion as written> — <test/scenario/command>
- [x] <criterion> — <evidence>. (Ticket said X; source has Y — corrected here.)
Kept for parity, beyond the ticket: <numbered list>
Deferred: <item> → <KEY>
Carried to: <KEY> — <one line>
```

The checkboxes in the ticket body are ticked as well (HTML `checked`), but only for
criteria verified in the PR.

---

## Changelog

- **v0 (2026-10-29)** — first draft from MFB-1587 review. Open: 3.1a, 3.5a, 3.7a, 3.9a, 4a.
