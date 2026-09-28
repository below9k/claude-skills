---

name: qa-ticket
description: QA a Jira ticket such as SN-#### against its acceptance criteria, implementation, and likely regression areas. Use when asked to QA, verify, validate, test, or regression-test a ticket before release.
context: fork
agent: qa-ticket
effort: high
------------

# QA Ticket

QA the ticket identified in the user's request.

The user's request is:

$ARGUMENTS

## Objective

Determine whether the implementation satisfies the ticket's acceptance criteria and whether it introduces any obvious regressions.

Do not implement fixes unless the user explicitly asks for them. This is an independent QA pass.

## Workflow

### 1. Retrieve the ticket

Use the configured Jira tools to retrieve the complete ticket.

Gather:

* ticket key
* title
* description
* acceptance criteria
* relevant comments
* linked tickets
* attachments when relevant
* current status
* referenced branch, merge request, pull request, commit, or release information

Read the actual acceptance criteria. Do not infer acceptance criteria from the ticket title alone.

If requirements are ambiguous, record the ambiguity as part of the QA findings rather than silently inventing expected behavior.

Determine whether meaningful acceptance criteria exist.

If the ticket has no acceptance criteria, or the existing criteria are insufficient to perform meaningful independent QA:

1. Invoke the `qa-acceptance-criteria` skill for this ticket.
2. Wait for the generated criteria to be posted to Jira.
3. Re-read the ticket so the QA pass uses the newly generated criteria.
4. Continue with the normal QA workflow.

Do not independently invent acceptance criteria inside this QA skill when the acceptance-criteria skill is available.

### 2. Identify the implementation

Determine what code implements the ticket.

Use, as available:

* branch names containing the ticket key
* GitLab/GitHub merge requests or pull requests
* commits referencing the ticket
* links from Jira
* git history
* current branch/worktree

Inspect the diff before testing.

Determine:

* what behavior changed
* which components/services are affected
* interfaces or APIs that changed
* configuration changes
* likely regression surfaces
* existing tests related to the changed behavior

Do not limit QA to the files named in the ticket.

### 3. Build a test plan

Create a concise test matrix before executing tests.

For every acceptance criterion, identify at least one verification.

Also consider:

* normal/expected use
* boundary conditions
* invalid or malformed input
* error handling
* state transitions
* persistence/reload behavior
* backward compatibility
* related existing behavior
* interactions with adjacent functionality
* likely regressions suggested by the diff

Prioritize tests based on risk rather than attempting arbitrary exhaustive coverage.

### 4. Execute verification

Use the appropriate tools available in the project.

This may include:

* existing unit tests
* integration tests
* end-to-end tests
* Playwright/browser automation
* API requests
* application logs
* build/typecheck/lint commands
* manual application interaction
* device or service interfaces available through configured tools

Prefer testing actual observable behavior over relying only on code inspection.

Code inspection alone does not prove an acceptance criterion passes when the behavior can reasonably be executed.

Do not modify production implementation code during QA.

Temporary test artifacts may be created when necessary, but clean them up afterward unless they provide useful permanent coverage and the user has explicitly asked for test implementation.

### 5. Investigate failures

For every failure:

1. reproduce it if possible
2. determine the conditions required to trigger it
3. collect useful evidence
4. identify the likely responsible code when reasonably possible
5. determine whether it is:

   * an acceptance-criteria failure
   * a regression
   * an unrelated pre-existing issue
   * an environment/test limitation
   * ambiguous because the requirement is unclear

Do not declare a defect solely because code looks suspicious.

Do not declare success for behavior that was not actually verified.

### 6. Update Jira

Post the QA result to the Jira ticket using the configured Jira tools.

Do not transition the ticket's status unless the user or project instructions explicitly require it.

The Jira update must contain a full QA writeup using this structure:

## QA Results

**Result:** PASS | PASS WITH NOTES | FAIL | BLOCKED

**Tested against:** <branch/MR/commit/build when known>

### Acceptance Criteria

* ✅ <criterion> — <how it was verified>
* ❌ <criterion> — <what failed>
* ⚠️ <criterion> — <limitation, ambiguity, or partial verification>

### Regression Testing

* ✅ <area tested> — <result>
* ❌ <area tested> — <failure>
* ⚠️ <area not fully testable> — <reason>

### Deficiencies

For each deficiency include:

**<short defect title>**

* Severity: Critical | High | Medium | Low
* Expected:
* Actual:
* Reproduction:
* Evidence:
* Likely area: <file/component when known>
* Acceptance criterion/regression affected:

If there are no deficiencies, explicitly state:

"No deficiencies were identified during this QA pass."

### Tests Executed

Include relevant commands, automated test suites, browser/device scenarios, API calls, and other verification performed.

Summarize large test-suite output rather than pasting it verbatim.

### Coverage / Limitations

State anything that could not be verified and why.

Examples:

* unavailable hardware
* inaccessible environment
* missing credentials
* external dependency unavailable
* acceptance criteria ambiguous
* test data unavailable

Never hide testing limitations.

### 7. Return the same result to the user

After updating Jira, provide the user with a concise summary containing:

* ticket
* overall QA result
* acceptance criteria result
* regression result
* deficiencies found
* anything blocked or not tested
* confirmation that the Jira ticket was updated

The Jira writeup should be the authoritative detailed report.

## Result definitions

Use these classifications consistently:

**PASS**
All acceptance criteria were verified successfully and no material regressions or deficiencies were found.

**PASS WITH NOTES**
Acceptance criteria pass, but there are minor observations, limitations, or unrelated issues worth documenting.

**FAIL**
At least one acceptance criterion fails or a regression attributable to the implementation was found.

**BLOCKED**
Meaningful QA cannot be completed because required infrastructure, hardware, credentials, build artifacts, requirements, or other dependencies are unavailable.

## QA principles

* Verify requirements, not assumptions.
* Inspect the implementation, but test behavior wherever practical.
* Test beyond the happy path.
* Look specifically for regressions in adjacent functionality.
* Separate confirmed defects from suspicions.
* Separate implementation defects from environment failures.
* Include reproducible evidence.
* Do not change implementation code to make QA pass.
* Do not mark untested behavior as passing.
* Do not omit failures from the Jira report.

