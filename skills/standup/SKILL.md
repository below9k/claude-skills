---
name: standup
description: >
  Use this skill whenever the user asks to prepare, generate, summarize, or
  review their standup or daily status update. Use Jira MCP and the forges (GitLab MCP, GitHub `gh`)
  to determine what the user actually worked on rather than relying only on
  conversation context. Review the user's recent Jira tickets, commits,
  branches, merge requests and pull requests, code reviews performed, review feedback addressed,
  QA activity, and other relevant development activity. By default, summarize
  work completed since the previous workday, identify work currently in
  progress or planned next, and surface blockers or items needing attention.
  Produce a concise standup suitable for sharing with the engineering team.
  Do not modify Jira, GitLab, or GitHub unless explicitly requested.
---

# Standup

## Purpose

Generate an accurate engineering standup based primarily on actual Jira and
forge activity (GitLab and GitHub), across every repo the user works in.

The standup should answer:

1. What did I work on?
2. What am I working on next?
3. Are there any blockers or issues?

Prefer concrete development activity over generic summaries.

Use Jira and the forges as evidence rather than inventing or assuming work.

---

## Default Time Window

Unless the user specifies another period, gather activity since the previous
workday.

For a normal weekday:

    Monday    -> since Friday
    Tuesday   -> since Monday
    Wednesday -> since Tuesday
    Thursday  -> since Wednesday
    Friday    -> since Thursday

Account for weekends when determining the previous workday.

If the user requests a different period, such as:

    "standup for yesterday"
    "what did I do this week?"
    "standup since Friday"

use that period instead.

---

# Phase 1 — Gather Jira Activity

Use Jira MCP to inspect tickets associated with the user's recent work.

Look for:

- Tickets assigned to the user.
- Tickets recently updated by the user.
- Tickets moved between statuses.
- Tickets currently in progress.
- Tickets completed since the previous standup period.
- Comments or QA updates made by the user.
- Relevant linked tickets.

For each relevant ticket, determine:

    Ticket
    Summary
    Current status
    Recent activity
    Relevant requirements/context

Do not include every assigned Jira ticket.

Include tickets only when they are relevant to the standup period or likely
to represent immediate upcoming work.

---

# Phase 2 — Gather Forge Activity

The user may work across several repos and trackers. The repo profiles in
`~/.claude/repos/*/README.md` list the known ones (forge, project path, Jira
site and project keys); check each, plus any repo the user names. Use the
GitLab MCP for GitLab projects and `gh` for GitHub (e.g.
`gh search prs --author @me --updated ">=<date>"`,
`gh search prs --reviewed-by @me --updated ">=<date>"`). In this skill, "MR"
means a GitLab merge request or a GitHub pull request.

Inspect the user's development activity during the standup
period.

Look for:

- Commits authored by the user.
- Branches worked on by the user.
- Merge requests created by the user.
- Merge requests updated by the user.
- Merge requests merged during the period.
- Code reviews performed by the user.
- Relevant MR discussions.
- Pipeline/build activity when meaningful.
- QA or fixes resulting from review feedback.

Pay particular attention to branches following each project's ticket naming
convention (from its profile), e.g.:

    PRJ-####/PRJ-###-<description>
    feat/prj-123-<description>

Use ticket identifiers to correlate forge activity with Jira.

Do not list individual commits when several commits represent the same piece
of work.

Summarize the resulting work instead.

For example, prefer:

    Finished the device reconnection changes for PRJ-567 and opened the MR.

rather than:

    Made 4 commits to PRJ-567.

---

# Phase 3 — Gather Code Review Activity

Use the forges to identify code review activity performed by the user during
the standup period.

Code review is meaningful engineering work and should be included in the
standup when substantial enough to mention.

Look for:

- Merge requests reviewed by the user.
- Review comments made by the user.
- Issues or defects identified during review.
- Suggestions or requested changes.
- Follow-up reviews after changes were made.
- Approval of merge requests.
- Discussions resolved by the user.
- Architecture or implementation feedback provided during review.
- Testing or validation performed as part of the review.

Determine what the review accomplished rather than merely reporting that an
MR was viewed.

For each relevant review, determine:

    MR
    Related Jira ticket, when available
    Purpose of the change
    Review performed
    Significant findings
    Current state

Summarize meaningful review outcomes.

Prefer:

    Reviewed the PRJ-742 MR and identified an issue with subscription cleanup
    during device disconnects.

or:

    Re-reviewed PRJ-742 after the requested changes and approved the MR.

over:

    Reviewed MR !123.

Do not include trivial interactions, such as simply opening an MR or making
a minor non-substantive comment.

If multiple review iterations occurred on the same MR, consolidate them into
one standup item when practical.

For example:

    Reviewed PRJ-742, requested changes around reconnect handling, and
    re-reviewed the updated implementation.

rather than reporting each review interaction separately.

## Reviews of the User's Own Work

Also identify meaningful review activity occurring on the user's merge
requests when it affected the user's work.

Examples include:

- Addressing reviewer feedback.
- Fixing defects found during review.
- Responding to implementation questions.
- Updating tests requested during review.
- Reworking an implementation based on review feedback.

Report the resulting engineering work rather than simply saying that review
comments were received.

Prefer:

    Addressed review feedback on PRJ-567 around error handling and added
    regression coverage.

over:

    Responded to comments on my MR.

## Code Review as Today's Work

When an MR is assigned to the user for review, has an outstanding review
request, or clearly requires another review pass, it may be included under
Today.

For example:

    Today:
    - Review the PRJ-810 device configuration MR.
    - Re-review PRJ-742 after the requested changes.

Do not assume every open MR is something the user intends to review today.
Use assignments, review requests, recent activity, or other available
evidence.

## Code Review Blockers

A code review may contribute to a blocker when there is concrete evidence
that progress cannot continue without action.

Examples:

- Required changes are waiting on another developer.
- An unresolved design question prevents approval.
- A failing pipeline prevents meaningful validation.
- The MR cannot proceed because requirements are unclear.

Normal pending review should not automatically be classified as a blocker.

---

# Phase 4 — Correlate Jira and the Forges

Combine Jira and forge information into logical work items.

For example:

    Jira PRJ-567
        +
    feature branch
        +
    commits
        +
    MR
        +
    review/fixes
        ↓
    One standup item

Avoid reporting the same work multiple times merely because it appears in
multiple systems.

Use Jira to understand what the work was intended to accomplish.

Use the forges to understand what development activity actually occurred.

---

# Phase 5 — Determine Completed / Recent Work

Identify meaningful work performed during the requested period.

Examples:

- Implemented a ticket.
- Finished a feature.
- Fixed a bug.
- Investigated a defect.
- Opened an MR.
- Updated an MR after review.
- Reviewed another developer's MR.
- Performed QA.
- Resolved test failures.
- Worked on infrastructure or tooling.
- Investigated a production/development issue.

Focus on outcomes rather than raw activity.

Prefer:

    Fixed the DSP subscription cleanup issue and opened the MR.

over:

    Modified three files and pushed two commits.

---

# Phase 6 — Determine Current / Next Work

Use Jira status, open branches, open MRs/PRs, and recent activity to
identify work likely to continue today.

Prioritize:

1. Work currently in progress.
2. Open MRs requiring changes.
3. Tickets actively assigned and started.
4. QA/review work that remains incomplete.
5. Clearly indicated next work.

Do not assume that every assigned backlog ticket is planned for today.

If the available information does not establish what the user intends to
work on next, say so rather than inventing a plan.

---

# Phase 7 — Identify Blockers

Look for evidence of blockers such as:

- Failed pipelines.
- Failing tests.
- Unresolved MR comments.
- Missing requirements.
- Jira tickets marked blocked.
- Dependency on another ticket.
- Dependency on another developer/team.
- Environment failures.
- Build/deployment problems.
- QA failures preventing completion.

Only report something as a blocker when it actually prevents or materially
delays progress.

Do not classify normal review or an open MR as a blocker merely because it
has not merged yet.

If no blockers are evident, report:

    No blockers.

---

# Phase 8 — Produce Standup

Keep the final standup concise.

Default format:

    Yesterday:
    - <meaningful work>
    - <meaningful work>

    Today:
    - <current/next work>
    - <current/next work>

    Blockers:
    - <blocker>

Use "Since last standup" instead of "Yesterday" when the time period spans
multiple days, such as Monday morning.

Example:

    Since last standup:
    - Finished PRJ-567 device reconnection handling and opened the MR.
    - Addressed review feedback on PRJ-482 and verified the changes.

    Today:
    - Continue PRJ-610 configuration synchronization work.
    - QA PRJ-567 after the latest build is available.

    Blockers:
    - No blockers.

---

# Writing Style

Standup output should sound like something the user would naturally say
during an engineering standup.

Keep it:

- Concise.
- Technical where useful.
- Outcome-focused.
- Specific.
- Easy to say aloud.

Avoid:

- Excessive implementation detail.
- Commit-by-commit summaries.
- Long Jira descriptions.
- Corporate language.
- Inflating routine work.
- Claiming something was completed when evidence only shows work in progress.

Ticket identifiers may be included when useful.

---

# Evidence and Uncertainty

Do not invent activity.

If Jira and a forge disagree, investigate the discrepancy when practical.

For example, a Jira ticket may still say "In Progress" even though its MR was
merged.

Use the available evidence to describe what actually happened without
silently changing Jira.

If uncertain, use language such as:

    Appears to still be in progress.

rather than presenting the conclusion as certain.

---

# Read-Only by Default

This skill is read-only by default.

Do not:

- Change Jira ticket status.
- Add Jira comments.
- Assign tickets.
- Modify merge requests.
- Merge branches.
- Push commits.
- Trigger pipelines.

unless the user explicitly requests the action.

The purpose of this skill is to gather activity and prepare the standup.

---

# Optional Detailed Mode

If the user asks for a detailed standup, also provide a private preparation
section containing:

    Ticket:
    Status:
    Forge activity (GitLab/GitHub):
    Current state:
    Next action:
    Potential blocker:

Keep this supporting detail separate from the concise standup that can be
read aloud.
