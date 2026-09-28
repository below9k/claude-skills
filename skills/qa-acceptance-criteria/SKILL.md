---

name: qa-acceptance-criteria
description: Generate missing acceptance criteria for a Jira ticket by analyzing the ticket, its implementation, and associated GitLab/GitHub MR or PR. Use when a ticket has no acceptance criteria or its acceptance criteria are insufficient for meaningful QA.
context: fork
agent: qa-acceptance-criteria
effort: high
------------

# Generate QA Acceptance Criteria

The user's request is:

$ARGUMENTS

Generate practical, testable acceptance criteria for the specified Jira ticket.

This skill is intended primarily for tickets that have no acceptance criteria or whose existing criteria are insufficient to perform meaningful QA.

## Objective

Create acceptance criteria that describe the behavior the completed ticket should provide, based on available evidence:

1. Jira ticket requirements
2. ticket comments and linked tickets
3. associated MR/PR
4. MR/PR description
5. implementation diff
6. existing tests
7. surrounding application behavior

Do not invent requirements that have no reasonable basis in the available evidence.

The criteria should be useful for a subsequent QA pass.

## Workflow

### 1. Retrieve the ticket

Retrieve the complete Jira ticket.

Inspect:

* title
* description
* comments
* linked tickets
* attachments
* related issues
* current status

Determine whether acceptance criteria already exist.

Treat criteria as missing when there are no meaningful, testable requirements.

If useful criteria already exist, do not replace them merely to improve wording.

### 2. Locate the implementation

Find the associated GitLab MR or GitHub PR.

Use:

* ticket key
* branch names
* MR/PR descriptions
* commit messages
* Jira links
* repository history

Inspect the complete diff.

Understand:

* what functionality was added or changed
* what behavior was removed
* affected APIs
* UI changes
* data/model changes
* configuration changes
* error handling
* dependencies
* affected integrations

### 3. Analyze surrounding behavior

Inspect relevant existing code and tests.

Determine:

* how the changed functionality is currently expected to behave
* how callers use the changed code
* existing validation rules
* existing error handling
* related workflows
* established conventions
* likely edge cases

Do not derive requirements solely from implementation details.

The goal is to describe externally observable behavior, not prescribe the implementation.

### 4. Generate acceptance criteria

Create a concise set of criteria.

Each criterion must be:

* specific
* observable
* testable
* independently verifiable
* relevant to the ticket
* grounded in evidence

Prefer approximately 3–8 criteria.

Avoid criteria that merely restate implementation details such as:

> "The developer adds a new function called `foo()`."

Prefer:

> "When the user performs X, the system produces Y."

Include appropriate criteria for:

* primary behavior
* important alternate paths
* validation
* error handling
* persistence/state
* compatibility
* relevant edge cases

Do not create unnecessary criteria simply to increase coverage.

### 5. Classify confidence

For every generated criterion, assign a confidence level:

**High**
Directly supported by the ticket description, explicit MR/PR requirements, or clearly documented existing behavior.

**Medium**
Strongly supported by the implementation and surrounding behavior but not explicitly stated.

**Low**
Reasonable inference necessary to make the behavior testable, but not clearly established by available evidence.

Avoid Low-confidence criteria when possible.

If a critical requirement cannot be established, call it out instead of presenting an assumption as fact.

### 6. Post to Jira

Post the generated criteria to the ticket.

Clearly identify them as **QA-derived acceptance criteria**.

Use this structure:

## QA-Derived Acceptance Criteria

The ticket did not contain sufficient acceptance criteria for independent QA verification. The following criteria were derived from the ticket requirements, associated MR/PR, implementation changes, and existing application behavior.

### Acceptance Criteria

* [ ] **AC1 — <short description>**

  * <testable requirement>
  * Confidence: High/Medium/Low
  * Basis: <brief evidence>

* [ ] **AC2 — <short description>**

  * <testable requirement>
  * Confidence: High/Medium/Low
  * Basis: <brief evidence>

...

### QA Notes

<Any ambiguities, assumptions, or requirements that could not be established from the available evidence.>

These criteria were generated for QA verification and should not be interpreted as additional product requirements unless confirmed by the appropriate product/engineering owner.

Do not modify the Jira ticket's original description.

Do not silently overwrite existing acceptance criteria.

### 7. Return the result

Return:

* ticket key
* number of criteria generated
* generated criteria
* confidence levels
* important assumptions/ambiguities
* MR/PR examined
* confirmation that the criteria were posted to Jira

## Important constraints

Never manufacture business requirements.

Never infer that a behavior is required solely because it is easy to test.

Never treat an implementation detail as a product requirement unless the implementation detail itself is externally observable and relevant.

If the ticket and implementation disagree, document the discrepancy.

If there is insufficient evidence to derive meaningful criteria, report that instead of creating speculative criteria.

