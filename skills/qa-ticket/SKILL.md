---
name: qa-ticket
description: >
  Use this skill when asked to QA, test, validate, or regression-test a Jira
  ticket or GitLab merge request in the project's deployed test environment.
  Use Jira MCP to retrieve requirements and acceptance criteria and GitLab
  MCP to inspect the MR, pipeline, changes, and artifacts. Deploy the latest
  appropriate artifact to the configured test server and test the running
  application. For user-facing functionality, perform live interactive
  browser testing using Playwright MCP: navigate the application, click
  controls, enter and edit values, submit forms, save changes, refresh,
  verify persistence, exercise relevant error states, and validate resulting
  backend or system behavior where applicable. Test as an actual user would,
  rather than only inspecting the DOM, screenshots, APIs, or source code.
  Perform regression testing of functionality affected by the changes and
  produce a detailed QA report.
context: fork
agent: qa-ticket
effort: high
---

# QA Ticket / Merge Request

QA the ticket or merge request identified in the user's request.

The user's request is:

$ARGUMENTS

## Purpose

Perform end-to-end QA of a Jira ticket or GitLab merge request on an actual
deployed build, validating both:

1. The implementation satisfies the ticket requirements.
2. The implementation does not introduce regressions in related existing
   functionality.

A successful build, installation, or unit-test run is not evidence that the
ticket works. This is an independent QA pass: do not implement fixes unless
the user explicitly asks.

## Inputs

The user may provide a Jira ticket (e.g. `SN-1234`), a GitLab MR, or both; a
release branch; a specific artifact; an instruction to test the latest
release candidate instead of the MR build; or an alternate test server.

Infer missing Jira/MR relationships through Jira and GitLab when they are
clearly linked. Never invent ticket numbers, MRs, release branches,
artifacts, or requirements.

## Default Test Server

    savis-101627.local            (SSH)
    http://savis-101627.local/    (web UI)

If the user specifies another test server, use it instead.

### Server Safety

Before installing an artifact or rebooting, confirm the SSH session is on the
intended test server with `hostname` and `hostname -f`. Never install or
reboot on a host whose identity you cannot establish confidently, and never
treat an SSH host as disposable merely because it is reachable.

---

# Phase 1 — Gather Requirements

Use the Jira MCP to retrieve the complete ticket: summary, description,
acceptance criteria (sometimes stored in `customfield_10160`), requirements,
comments, linked issues, relevant attachments, status, reproduction steps,
and any referenced branch, MR, commit, or release.

Read the actual acceptance criteria; never infer them from the title. Where
later comments clarify or supersede the original description, prefer the
most recent explicit clarification. Record ambiguities as QA findings rather
than inventing expected behavior.

## Missing Acceptance Criteria

If the ticket has no acceptance criteria, or they are too vague for
meaningful independent QA, derive them before testing. This skill runs as a
subagent and cannot start another, so do it here:

1. Read `~/.claude/skills/qa-acceptance-criteria/SKILL.md`.
2. Follow its workflow and rules — evidence hierarchy, no requirement
   inflation, a confidence level and basis per criterion — and post the
   "QA-Derived Acceptance Criteria" comment to Jira in its format.
3. Use those criteria as the test plan, and note in the QA report that they
   were QA-derived.

## Test Checklist

For each requirement, write down:

    Requirement → Expected behavior → Test procedure (including the user
    interactions to perform) → Expected result

---

# Phase 2 — Inspect the GitLab Change

Use the GitLab MCP to inspect the MR: source and target branches, commits,
the complete diff, description, pipeline status, discussions, available
artifacts, and existing tests for the changed behavior.

The target is normally `release/v#.#.#` or `release/<project>/v#.#.#`; never
assume `main`, `master`, or `develop`.

Compare the implementation against the Jira requirements and determine what
behavior changed, which components, services, interfaces, APIs, and
configuration are affected, and where regressions are likely. Do not limit
QA to the files named in the ticket. This sets the regression scope for
Phase 13.

---

# Phase 3 — Select the Artifact

## Default: MR Artifact

Unless instructed otherwise, use the latest successful build artifact for
the MR, preferably from the latest successful pipeline on the current MR HEAD
commit. Verify that the pipeline succeeded, and that the artifact belongs to
the expected branch/MR and commit, has not expired, and is installable on the
test server.

Never silently test an artifact from an older MR commit when a newer commit
exists.

Record:

    MR:
    Commit:
    Pipeline:
    Job:
    Artifact:
    Build/version:

## Release Candidate Mode

Only if the user explicitly asks to test the latest RC: determine the MR's
release branch, find its latest successful RC artifact in GitLab, verify the
artifact actually belongs to that release, and record the exact RC/build
version. Never substitute an RC for the MR artifact otherwise.

---

# Phase 4 — Download Artifact

Download the artifact with the GitLab MCP or other configured GitLab
tooling. Confirm the download completed, verify checksums when available,
and do not proceed with an incomplete or corrupt artifact.

---

# Phase 5 — Upload to Test Server

Upload to `/tmp` on the test server, using a unique path such as
`/tmp/<artifact-name>` so it cannot be confused with earlier QA artifacts.
Confirm the upload completed before installing.

---

# Phase 6 — Install

SSH in and verify the server identity again. In `/tmp`, extract the artifact
in its actual format — inspect it rather than assuming; for a tar archive
this is normally `tar -xf <artifact>`.

Run the installer packaged with the artifact, or the one the repository
documents, with the privileges the normal installation requires. Capture its
output and exit status.

If installation fails: capture the error, determine whether the artifact,
environment, or procedure is at fault, and report it if it cannot be safely
resolved. Never reboot to hide or recover from an unexplained failed
installation.

---

# Phase 7 — Reboot

After a successful installation, and only after verifying the connected
machine is the intended test server, reboot it:

    reboot -f

Use elevated privileges if required. The SSH connection will drop; that is
expected, not a test failure.

---

# Phase 8 — Wait for Recovery

Wait for the server to become reachable, reconnect, and check `hostname` and
`uptime` to confirm it actually rebooted. Confirm the required application
services have started, checking service status and logs as needed. Do not
begin functional testing until the system is initialized enough for results
to be meaningful.

---

# Phase 9 — Verify Installed Build

Confirm the installed build matches the selected artifact, using the
application's version information, package metadata, service information,
UI version, or another reliable source.

    Expected build:
    Installed build:

If they differ, stop and investigate. Never QA an unknown build.

---

# Phase 10 — Ticket Acceptance Testing

Execute the Phase 1 checklist, recording for each requirement:

    Requirement:
    Test:
    Expected:
    Actual:
    Result: PASS | FAIL | BLOCKED

Test actual behavior. Code or configuration inspection does not prove a
criterion passes when the behavior can reasonably be executed. Use the
browser for user-facing functionality (Phase 11) and SSH, logs, APIs, and
services for backend and system behavior (Phase 12). When a feature crosses
layers, validate the complete behavior, not one layer.

---

# Phase 11 — Live Interactive UI Testing

For any ticket that affects user-facing functionality, test the deployed
application interactively using the configured browser automation,
preferably Playwright MCP, at `http://savis-101627.local/` or the alternate
server. Authenticate only with test credentials that are already configured
and authorized.

Loading a page, inspecting the DOM, reading page text, looking at
screenshots, calling APIs directly, reviewing code, or confirming that
elements exist is **not** UI testing. Interact as a real user would, as
relevant to the ticket:

- Navigate the UI; open pages, dialogs, menus, tabs, and panels.
- Click buttons and controls, select options, enter and edit values.
- Submit forms and save configuration; cancel operations and verify the
  cancellation.
- Refresh, navigate away and return, and verify persisted values after
  reload.
- Exercise validation and error states, and repeat operations where
  relevant.
- Verify enabled/disabled states, state transitions, loading and
  reconnection behavior, and the feedback shown to the user.
- Check the browser console for errors and network requests for failures.
- Delete test-created data when safe.
- Verify that actions actually affect the underlying system.

Reproduce the Jira workflow from the user's perspective. If Jira says a user
can modify a setting, do not stop at confirming the input exists:

    Open page → locate setting → record original value → change it → save
      → verify success → refresh or revisit → verify it persisted
      → verify resulting system behavior → restore original value

If a ticket adds a button:

    Navigate to feature → verify initial state → click → observe behavior
      → verify UI state → verify backend/system effect

Never mark a user-facing criterion PASS without exercising the interaction.
If testing is blocked, mark it BLOCKED.

Do not add permanent test infrastructure during a QA run. If a permanent
automated regression test would be valuable, recommend it instead.

## Verify Effects Beyond the UI

When a UI action should change backend, device, configuration, or system
state, verify both sides when practical:

    UI action → UI indicates success → API/network behavior
      → server/device/system state → UI reflects resulting state

A success notification alone is not evidence the operation succeeded. Use
SSH, logs, APIs, device state, or other available mechanisms.

## Test Both Positive and Negative Paths

Test the successful workflow and the relevant failure and boundary
behavior: valid, invalid, and empty input; minimum and maximum values;
repeated submission; cancellation; refresh during or after an operation;
navigating away and back; failed backend requests when reasonably
reproducible; and disconnected or unavailable devices when relevant. Only
test scenarios relevant to the change that can be performed safely.

## Preserve the Test Environment

Record the original state before changing existing configuration or data,
and restore anything changed solely for QA unless the test requires the
result to remain. Clean up test-created data when practical. Never delete or
destructively modify unrelated user, project, device, or system data.

## Evidence

For each criterion, record the interactions performed and the observed
result. For failures, capture the exact workflow, visible error, console
errors, failed network requests, a screenshot when supported, relevant
server/application logs, and the resulting system state.

Report what was actually exercised, never just that something was "tested."

---

# Phase 12 — System Testing

Use SSH to validate system behavior as the ticket requires: running services
and process state, logs, configuration, network connectivity, device
communication, file and database/service state, resource usage, restart
behavior, and persistence across reboot.

For backend/API changes, exercise the affected endpoint or service directly
when practical: successful and error responses, payloads, and
authentication/authorization.

Prefer observable behavior over assumptions from the implementation.

---

# Phase 13 — Regression Testing

Regression testing is risk-based, not an arbitrary full-system test. From
the diff and the Jira requirements, trace:

    Changed component → dependencies/consumers → existing behaviors
      → regression tests

Focus on callers of changed functions; shared modules and components;
existing workflows through changed code; APIs, contracts, and schemas; state
management; device communication; authentication/authorization;
persistence; configuration; startup/shutdown; async behavior and retries;
reconnection; error handling; and features adjacent to the ticket.

Exercise user-facing regression areas interactively, as in Phase 11. Test
existing behavior both inside and immediately outside the ticket's path.

Passing acceptance criteria is not evidence of "no regressions."

---

# Phase 14 — Investigate Failures

For every failure: reproduce it, determine whether it is repeatable and what
triggers it, check whether it pre-dates the ticket when practical, collect
evidence from the browser, console, network, application and service logs,
system state, the GitLab diff, and Jira, and identify the likely responsible
code when the evidence supports it.

Classify it as one of: product defect or acceptance-criteria failure,
regression, environment problem, installation/deployment problem, test
problem, ambiguous requirement, or pre-existing behavior.

Do not declare a defect because code looks suspicious, and do not present
speculation as fact. Do not modify application code to make QA pass unless
the user explicitly asked this workflow to fix defects.

---

# Phase 15 — Retesting

When a failure is resolved during the session, repeat the failing test, the
closely related tests, and the relevant regression tests. A disappeared
error alone does not mean the issue is resolved.

---

# Phase 16 — Update Jira

Post one comprehensive QA report to the Jira ticket using the Jira MCP. Do
not transition the ticket's status unless the user or project instructions
require it. Do not post speculative findings as confirmed defects.

    ## QA Results

    **Result:** PASS | PASS WITH NOTES | FAIL | BLOCKED

    ### Build

    Jira:
    MR:
    Base:
    Commit:
    Pipeline:
    Artifact:
    Installed version:
    Server:

    ### Acceptance Criteria
    (Note if the criteria were QA-derived.)

    [PASS] <criterion>
    Interactions performed:
    Expected:
    Actual:

    [FAIL] <criterion>
    Interactions performed:
    Expected:
    Actual:

    [BLOCKED] <criterion>
    Reason:

    ### Regression Testing

    [PASS] <existing behavior> — <what was exercised>
    [FAIL] <existing behavior> — <failure>
    [BLOCKED] <behavior> — <reason>

    ### Deficiencies

    #### <severity: Critical | High | Medium | Low> — <description>

    Steps to reproduce:
    1.
    2.
    3.

    Expected:
    Actual:
    Evidence: <logs/errors/console/network/browser observations>
    Affected criterion/area:
    Likely code area: <file/component when known>

    If none were found:
    "No deficiencies were identified during this QA pass."

    ### Coverage

    Tested:
    -

    Not tested (and why):
    -

    ### Summary

    Requirements: <passed>/<total>
    Regression checks: <passed>/<total>

Summarize large logs rather than pasting them. State every testing
limitation — unavailable hardware, inaccessible environment, missing
credentials, unavailable dependencies, ambiguous criteria, missing test
data.

## Optional GitLab Update

If the user asks, post implementation-specific findings to the MR. Jira
remains the broader QA record.

---

# Phase 17 — Return the Result to the User

After updating Jira, give the user a concise summary: ticket and MR, build
and server tested, overall result, acceptance-criteria and regression
results, deficiencies found, anything blocked or untested, and confirmation
that Jira was updated. The Jira report is the authoritative detailed record.

---

## Result Definitions

**PASS** — All acceptance criteria verified; no material regressions or
deficiencies found.

**PASS WITH NOTES** — Acceptance criteria pass, with minor observations,
limitations, or unrelated issues worth documenting.

**FAIL** — At least one acceptance criterion fails, or a regression
attributable to the implementation was found.

**BLOCKED** — Meaningful QA cannot be completed because required
infrastructure, hardware, credentials, artifacts, requirements, or other
dependencies are unavailable.

Never report PASS if a required criterion failed or required testing was
blocked. Always distinguish untested behavior from passing behavior.

---

# Safety Rules

Never:

- Install an unverified artifact, or install on an unidentified SSH host.
- Reboot a host not verified as the intended test server.
- Test an old MR artifact while reporting it as the latest.
- Treat a successful pipeline, installation, rendered UI, or success
  notification as proof the feature works.
- Invent Jira requirements or test results, or report untested behavior as
  passing.
- Hide failures by changing the test procedure, modifying implementation
  code, weakening tests, or altering acceptance criteria.
- Delete or destructively modify unrelated user, project, device, or system
  data.
- Omit failures from the Jira report.
- Use production systems unless the user explicitly names them as the
  intended environment.

The default deployment target is the dedicated test server
`savis-101627.local`.
