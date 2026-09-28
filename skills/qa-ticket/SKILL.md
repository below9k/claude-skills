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

Perform end-to-end QA of a Jira ticket or GitLab merge request using an
actual deployed build.

This skill should validate both:

1. The implementation satisfies the ticket requirements.
2. The implementation does not introduce regressions in related existing
   functionality.

Do not treat a successful build, successful installation, or passing unit
tests as sufficient evidence that the ticket works correctly.

Do not implement fixes unless the user explicitly asks for them. This is an
independent QA pass.

---

## Inputs

The user may provide:

- Jira ticket, such as `SN-1234`.
- GitLab merge request.
- Both Jira ticket and MR.
- Release branch.
- Specific artifact.
- Instruction to test the latest release candidate instead of the MR build.
- Alternate test server.

Infer missing Jira/MR relationships through Jira and GitLab when they are
clearly linked.

Do not invent ticket numbers, merge requests, release branches, artifacts,
or requirements.

---

## Default Test Server

The normal QA server is:

    savis-101627.local

Normal web interface:

    http://savis-101627.local/

The server is expected to be accessible over SSH.

If the user specifies another test server, use that server instead.

### Server Safety

Before installing an artifact or rebooting the server, verify that the SSH
connection is to the intended test server.

Inspect identifying information such as:

    hostname
    hostname -f

The hostname should correspond to the configured test server.

Never run installation or reboot commands on a server whose identity cannot
be established confidently.

Never infer that an arbitrary SSH host is disposable simply because it is
reachable.

---

# Phase 1 — Gather Requirements

Use the Jira MCP to retrieve the complete ticket.

Review:

- Summary.
- Description.
- Acceptance criteria (sometimes stored in `customfield_10160`).
- Requirements.
- Relevant comments.
- Linked issues when relevant.
- Attachments when relevant.
- Current status.
- Reported bugs or reproduction steps.
- Referenced branch, merge request, commit, or release information.
- Clarifications added after the original description.

Read the actual acceptance criteria. Do not infer acceptance criteria from
the ticket title alone.

Prefer the most recent explicit Jira clarification when ticket comments
clarify or supersede older requirements.

If requirements are ambiguous, record the ambiguity as part of the QA
findings rather than inventing expected behavior.

## Missing Acceptance Criteria

If the ticket has no acceptance criteria, or the existing criteria are
insufficient to perform meaningful independent QA:

1. Invoke the `qa-acceptance-criteria` skill for this ticket.
2. Wait for the generated criteria to be posted to Jira.
3. Re-read the ticket so the QA pass uses the newly generated criteria.
4. Continue with the normal QA workflow.

Do not independently invent acceptance criteria inside this QA skill when
the acceptance-criteria skill is available.

## Test Checklist

Build a concrete test checklist from the requirements.

For each requirement, determine:

    Requirement
        ↓
    Expected behavior
        ↓
    Test procedure (including the user interactions to perform)
        ↓
    Expected result

---

# Phase 2 — Inspect the GitLab Change

Use the GitLab MCP to inspect the associated merge request.

Determine:

- Source branch.
- Target/base branch.
- Commits.
- Changed files and the complete diff.
- MR description.
- Pipeline status.
- Relevant discussions.
- Available artifacts.
- Existing tests related to the changed behavior.

The target branch is commonly:

    release/v#.#.#

or:

    release/PRJ/v#.#.#

Do not assume `main`, `master`, or `develop` is the base.

Compare the implementation against the Jira requirements.

Determine:

- What behavior changed.
- Which components/services are affected.
- Interfaces or APIs that changed.
- Configuration changes.
- Likely regression surfaces.

Do not limit QA to the files named in the ticket.

Use this information to create the regression test scope.

---

# Phase 3 — Select the Artifact

## Default: MR Artifact

Unless instructed otherwise, select the latest successful build artifact
associated with the merge request being tested.

Prefer an artifact from the latest successful pipeline for the current MR
HEAD commit.

Verify:

- Pipeline succeeded.
- Artifact belongs to the expected branch/MR.
- Artifact corresponds to the expected commit.
- Artifact has not expired.
- Artifact is an installable build appropriate for the test server.

Do not silently test an artifact from an older MR commit when a newer commit
exists.

Record:

    MR:
    Commit:
    Pipeline:
    Job:
    Artifact:
    Build/version:

## Release Candidate Mode

If the user explicitly asks to test the latest RC, determine the release
branch on which the ticket/MR is based.

Typical branches:

    release/v#.#.#
    release/PRJ/v#.#.#

Use GitLab to identify the latest appropriate successful RC artifact for
that release.

Verify that the selected artifact actually belongs to the intended release.

Record the exact RC/build version being tested.

Do not substitute an RC build for the MR artifact unless the user requested
RC testing.

---

# Phase 4 — Download Artifact

Download the selected artifact from GitLab using the GitLab MCP or other
configured GitLab tooling.

Verify that the download completed successfully.

When checksums or other artifact integrity information are available, verify
them.

Do not proceed with an obviously incomplete or corrupt artifact.

---

# Phase 5 — Upload to Test Server

Upload the artifact to:

    /tmp

on the configured test server.

Prefer a unique path when necessary to avoid confusing the new artifact with
previous QA artifacts.

Example:

    /tmp/<artifact-name>

Verify that the upload completed successfully before installation.

---

# Phase 6 — Install

SSH into the test server.

Verify the server identity again before performing installation.

Move to `/tmp` and extract the artifact using the format appropriate to the
artifact.

For a tar archive, this will normally resemble:

    cd /tmp
    tar -xf <artifact>

Do not assume the archive format. Inspect it when necessary.

Locate the artifact's installation script/file.

Prefer the installation procedure packaged with the artifact or documented
by the repository.

Run the installer with the privileges required by the project's normal
installation process.

Capture installation output and check its exit status.

If installation fails:

1. Capture the relevant error.
2. Determine whether the failure is caused by the artifact, environment, or
   installation procedure.
3. Do not reboot merely to hide or recover from an unexplained failed
   installation.
4. Report the failure if it cannot be safely resolved.

---

# Phase 7 — Reboot

After a successful installation, reboot the TEST SERVER:

    reboot -f

Use elevated privileges when required by the server.

This command is permitted only after verifying that the connected machine is
the intended QA/test server.

Expect the SSH connection to terminate.

Do not treat the resulting disconnect as a test failure.

---

# Phase 8 — Wait for Recovery

Wait for the test server to become reachable again.

Reconnect using SSH.

Verify:

    hostname
    uptime

Confirm that the server actually rebooted.

Verify that required application services have started.

Inspect relevant service status and logs when appropriate.

Do not begin functional testing until the system is sufficiently initialized
to provide meaningful results.

---

# Phase 9 — Verify Installed Build

Before functional testing, confirm that the expected build was installed.

Use the application's available version information, package metadata,
service information, UI version, or other reliable source.

Compare the installed version against the artifact selected earlier.

Do not perform QA against an unknown build.

Record:

    Expected build:
    Installed build:

If they do not match, stop and investigate before testing the ticket.

---

# Phase 10 — Ticket Acceptance Testing

Execute the test checklist derived from Jira.

For each requirement record:

    Requirement:
    Test:
    Expected:
    Actual:
    Result: PASS | FAIL | BLOCKED

Test actual behavior rather than merely inspecting configuration or code.

Code inspection alone does not prove an acceptance criterion passes when the
behavior can reasonably be executed.

Use the browser for user-facing functionality (Phase 11).

Use SSH, logs, APIs, services, or other appropriate mechanisms for backend
and system functionality (Phase 12).

When the feature crosses multiple layers, validate the complete behavior
rather than testing only one layer.

---

# Phase 11 — Live Interactive UI Testing

For any ticket that affects user-facing functionality, perform live,
interactive testing against the deployed application using the configured
browser automation tooling, preferably Playwright MCP, at:

    http://savis-101627.local/

or the alternate server specified by the user.

Authenticate only with test credentials that are already configured and
authorized.

Do not limit browser testing to:

- Checking whether the page loads.
- Inspecting DOM elements.
- Reading page text.
- Inspecting screenshots.
- Calling APIs directly.
- Reviewing the implementation.
- Verifying that elements merely exist.

Interact with the application as a real user would.

This includes, when applicable:

- Navigate through the application's UI.
- Open pages, dialogs, menus, tabs, and panels.
- Click buttons and controls.
- Select options.
- Enter and edit values.
- Submit forms.
- Save configuration changes.
- Cancel operations and verify cancellation behavior.
- Delete or remove test-created data when safe.
- Refresh the page.
- Navigate away and return.
- Verify persisted values after reload.
- Exercise validation and error states.
- Perform repeated operations when relevant.
- Verify enabled/disabled states.
- Verify state transitions.
- Verify loading behavior.
- Verify reconnection behavior when relevant.
- Verify feedback shown to the user.
- Check the browser console for errors.
- Check network requests for failures.
- Verify that actions actually affect the underlying system.

The browser should be used to reproduce the workflow described by the Jira
ticket from the perspective of an actual user.

For example, if Jira requires that a user can modify a setting, do not merely
verify that the setting's input exists.

Perform the workflow:

    Open relevant page
        ↓
    Locate setting
        ↓
    Record original value
        ↓
    Change value
        ↓
    Save/submit
        ↓
    Verify success
        ↓
    Refresh or revisit page
        ↓
    Verify value persisted
        ↓
    Verify resulting system behavior when applicable
        ↓
    Restore original value when appropriate

Similarly, if a ticket adds a button:

    Navigate to feature
        ↓
    Verify appropriate initial state
        ↓
    Click button
        ↓
    Observe resulting behavior
        ↓
    Verify UI state
        ↓
    Verify backend/system effect when applicable

Do not mark a user-facing acceptance criterion PASS without exercising the
relevant interaction unless testing is blocked.

If testing is blocked, report the criterion as BLOCKED rather than PASS.

Do not create permanent test infrastructure during a normal QA run. If a
permanent automated regression test appears appropriate, report that
opportunity rather than silently adding it.

## Verify Effects Beyond the UI

When a UI action is expected to cause a backend, device, configuration, or
system change, verify both sides when practical.

For example:

    UI action
        ↓
    UI indicates success
        ↓
    Verify API/network behavior
        ↓
    Verify server/device/system state
        ↓
    Verify UI reflects resulting state

A success notification alone is not sufficient evidence that the underlying
operation succeeded.

Use SSH, logs, APIs, device state, or other available mechanisms to verify
the resulting system behavior when relevant.

## Test Both Positive and Negative Paths

For changed functionality, test the normal successful workflow and relevant
failure or boundary behavior.

Examples include:

- Valid input.
- Invalid input.
- Empty input.
- Minimum/maximum values.
- Repeated submission.
- Cancel behavior.
- Refresh during or after an operation.
- Navigation away and back.
- Failed backend requests when reasonably reproducible.
- Disconnected/unavailable devices when relevant.

Only test scenarios that are relevant to the change and can be performed
safely.

## Preserve the Test Environment

Before modifying existing configuration or data through the UI, record the
original state when practical.

After testing, restore configuration or data that was changed solely for QA,
unless the test specifically requires the resulting state to remain.

Do not delete or destructively modify unrelated user, project, device, or
system data.

Test-created temporary data should be cleaned up when practical.

## Evidence

For each acceptance criterion, record the meaningful interactions performed
and observed result.

When a failure occurs, capture useful evidence such as:

- Exact workflow performed.
- Visible error.
- Browser console error.
- Failed network request.
- Relevant screenshot when supported.
- Relevant server/application logs.
- Resulting system state.

The QA report should describe what was actually exercised rather than simply
stating that the feature was "tested."

---

# Phase 12 — System Testing

Use SSH when necessary to validate system behavior.

Depending on the ticket, inspect:

- Running services.
- Process state.
- Logs.
- Configuration.
- Network connectivity.
- Device communication.
- File state.
- Database/service state.
- Resource usage.
- Restart behavior.
- Persistence across reboot.

For backend/API changes, exercise the affected endpoint or service directly
when practical: verify successful responses, relevant error responses,
payloads, and authentication/authorization behavior where applicable.

Prefer observable behavior over assumptions based solely on implementation.

---

# Phase 13 — Regression Testing

Use the GitLab diff and Jira requirements to determine what existing
functionality could plausibly be affected.

Regression testing should be risk-based rather than an arbitrary full-system
test.

Identify:

    Changed component
          ↓
    Dependencies / consumers
          ↓
    Existing behaviors
          ↓
    Regression tests

Pay particular attention to:

- Callers of changed functions.
- Shared modules and components.
- Existing workflows using changed code.
- APIs whose behavior or contracts changed.
- Schemas.
- State management.
- Device communication.
- Authentication/authorization.
- Persistence.
- Configuration.
- Startup/shutdown behavior.
- Asynchronous behavior and retries.
- Reconnection behavior.
- Error handling.
- Features adjacent to the ticket's functionality.

Exercise user-facing regression areas interactively in the browser, as in
Phase 11.

Test important existing behavior both inside and immediately outside the
ticket's intended path.

Do not report "no regressions" merely because the ticket's acceptance
criteria pass.

---

# Phase 14 — Investigate Failures

For every failure:

1. Reproduce it if possible.
2. Determine whether it is repeatable and the conditions required to trigger
   it.
3. Determine whether it existed before the ticket when practical.
4. Collect evidence.
5. Identify the likely responsible code when supported by evidence.

Distinguish among:

1. Product defect / acceptance-criteria failure.
2. Regression.
3. Environment problem.
4. Installation/deployment problem.
5. Test problem.
6. Ambiguous requirement.
7. Pre-existing behavior.

Use:

- Browser behavior.
- Browser console.
- Network requests.
- Application logs.
- Service logs.
- System state.
- GitLab changes.
- Jira requirements.

Do not declare a defect solely because code looks suspicious.

Do not present speculation as fact.

Do not modify production/application code merely to make QA pass unless the
user has explicitly asked this workflow to fix discovered defects.

The primary purpose of this skill is to test and report.

---

# Phase 15 — Retesting

When a failure is resolved during the QA session:

1. Repeat the failing test.
2. Repeat closely related tests.
3. Repeat relevant regression tests.

Do not mark an issue resolved solely because the immediate error disappeared.

---

# Phase 16 — Update Jira

Post one comprehensive QA report to the Jira ticket using the Jira MCP.

Do not transition the ticket's status unless the user or project
instructions explicitly require it.

Do not post speculative findings as confirmed defects.

Use this structure:

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

Summarize large logs rather than pasting them verbatim.

Never hide testing limitations such as unavailable hardware, inaccessible
environment, missing credentials, unavailable external dependencies,
ambiguous acceptance criteria, or missing test data.

## Optional GitLab Update

If requested by the user, post relevant QA findings to the GitLab merge
request.

Keep GitLab comments focused on implementation-specific findings.

Use Jira for the broader QA record.

---

# Phase 17 — Return the Result to the User

After updating Jira, provide the user with a concise summary containing:

- Ticket and MR.
- Build/artifact tested and server.
- Overall QA result.
- Acceptance criteria result.
- Regression result.
- Deficiencies found.
- Anything blocked or not tested.
- Confirmation that the Jira ticket was updated.

The Jira writeup is the authoritative detailed report.

---

## Result Definitions

**PASS**
All acceptance criteria were verified successfully and no material
regressions or deficiencies were found.

**PASS WITH NOTES**
Acceptance criteria pass, but there are minor observations, limitations, or
unrelated issues worth documenting.

**FAIL**
At least one acceptance criterion fails or a regression attributable to the
implementation was found.

**BLOCKED**
Meaningful QA cannot be completed because required infrastructure, hardware,
credentials, build artifacts, requirements, or other dependencies are
unavailable.

Do not report PASS if a required acceptance criterion failed.

Do not report PASS if required testing was blocked.

Clearly distinguish untested behavior from passing behavior.

---

# Safety Rules

Never:

- Install an unverified artifact.
- Install on an unidentified SSH host.
- Reboot a host that has not been verified as the intended test server.
- Test an old MR artifact while reporting it as the latest.
- Assume a successful pipeline means the feature works.
- Assume successful installation means the feature works.
- Assume a rendered UI or success notification means the feature works.
- Invent Jira requirements.
- Invent test results.
- Report untested behavior as passing.
- Hide failures by modifying the test procedure.
- Modify implementation code, weaken tests, or alter acceptance criteria to
  make QA pass.
- Delete or destructively modify unrelated user, project, device, or system
  data.
- Omit failures from the Jira report.
- Use production systems for this workflow unless the user explicitly
  identifies them as the intended environment.

The default deployment target for this skill is the dedicated test server:

    savis-101627.local
