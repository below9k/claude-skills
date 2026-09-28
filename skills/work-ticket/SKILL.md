---

name: work-ticket
description: Implement an engineering ticket such as SN-#### from start to finish. Use when asked to work, implement, fix, develop, or complete a Jira ticket. Retrieves the ticket, reads repository Claude instructions, identifies the associated code and branch, implements the requested behavior, verifies it, iterates on failures, and reports completion.
context: fork
agent: work-ticket
effort: high
------------

# Work Ticket

Work the engineering ticket identified in:

$ARGUMENTS

## Objective

Take the ticket from requirements to a complete, locally verified implementation that is ready for review.

The goal is not merely to produce code. The completed work should:

* satisfy the ticket requirements
* follow repository conventions
* preserve existing behavior unless intentionally changed
* include appropriate tests
* pass relevant validation
* avoid unnecessary scope
* be understandable and reviewable

## Workflow

### 1. Retrieve and understand the ticket

Retrieve the complete ticket using the configured Jira tools.

Read:

* title
* description
* acceptance criteria
* additional information
* what do you want added/removed?
* how should it behave?
* attachments
* comments
* linked tickets
* attachments
* related work
* current status
* referenced branches, commits, MRs, or PRs

Identify:

* requested behavior
* explicit acceptance criteria
* constraints
* known edge cases
* affected systems
* unresolved ambiguity

If the ticket is a bug check the fields:

* what is broken?
* what will it do/look like to be unbroken?

Do not begin implementation based only on the ticket title.

If requirements are ambiguous, first investigate the repository, related tickets, existing behavior, and implementation context.

Only block on ambiguity when materially different implementations remain possible and there is not enough evidence to choose safely.

### 2. Read repository instructions

Before planning implementation, discover and read the repository's existing Claude instructions.

At minimum inspect when present:

* `CLAUDE.md`
* `.claude/CLAUDE.md`
* parent-directory `CLAUDE.md` files
* directory-specific `CLAUDE.md` files relevant to files being modified
* `.claude/rules/`
* repository-specific skills relevant to the work

Also inspect relevant developer documentation such as:

* `README.md`
* contribution documentation
* architecture documentation
* package scripts
* test documentation

Repository instructions override generic workflow preferences in this skill when they conflict.

Do not assume project commands, architecture, branching conventions, testing procedures, or code style when the repository already documents them.

### 3. Establish repository state

Before making changes:

* inspect the current branch
* inspect working-tree status
* identify existing uncommitted changes
* determine whether those changes belong to the user or the ticket
* locate the ticket's branch or associated MR/PR if one already exists
* inspect relevant git history

Never overwrite or discard unrelated user changes.

If the repository's documented workflow requires a ticket branch, follow it.

If an existing branch or MR/PR represents the ticket, continue from that work rather than creating a parallel implementation unless repository instructions say otherwise.

### 4. Investigate before implementing

Inspect the code paths relevant to the ticket.

Read enough surrounding code to understand:

* current behavior
* architecture
* ownership boundaries
* data flow
* callers and consumers
* existing abstractions
* relevant types/interfaces
* error handling
* existing tests
* related historical implementation where useful

Search for similar existing patterns before introducing new ones.

Do not speculate about code you have not inspected.

### 5. Determine scope and plan

Create a concise implementation plan grounded in:

* ticket requirements
* acceptance criteria
* repository instructions
* existing architecture
* existing patterns

Identify:

* code that needs to change
* tests that need to change or be added
* likely regression areas
* validation required before completion

Prefer the smallest implementation that completely satisfies the ticket.

Do not add unrelated refactors or improvements.

### 6. Implement incrementally

Make focused changes in logical increments.

Prefer:

* existing abstractions
* existing patterns
* existing dependencies
* established naming conventions
* established error-handling patterns
* DRY (Don't Repeat Yourself)

Avoid:

* unnecessary abstractions
* unrelated cleanup
* speculative extensibility
* dependencies that are not necessary
* broad rewrites when a targeted change is sufficient
* over-engineering

Preserve backward compatibility unless the ticket explicitly changes existing behavior.

### 7. Add or update tests

When the repository has automated testing appropriate to the change, add or update tests that verify the requested behavior.

Tests should cover the important behavior introduced or modified by the ticket.

Where appropriate include:

* expected behavior
* relevant boundary conditions
* failure/error behavior
* regression coverage

Do not modify tests merely to accommodate incorrect implementation behavior.

Do not remove meaningful assertions simply to make tests pass.

Prefer testing observable behavior over implementation details.

### 8. Verify continuously

Do not wait until the entire implementation is complete before checking it.

After meaningful changes, run the narrowest useful validation.

Examples:

* targeted unit tests
* affected integration tests
* type checking
* linting
* builds
* targeted runtime checks

Use failures as feedback for the next implementation iteration.

### 9. Iteration loop

When validation fails or behavior does not match expectations:

1. inspect the failure
2. determine whether the failure comes from:

   * implementation
   * test expectation
   * environment
   * pre-existing behavior
   * misunderstood requirement
3. inspect the relevant code and runtime evidence
4. revise the implementation or test when justified
5. re-run the narrowest relevant validation
6. continue until the issue is resolved or demonstrably blocked

Do not blindly patch symptoms one at a time.

When repeated attempts fail, reconsider the underlying assumption or design rather than repeatedly modifying the same area.

Do not weaken tests to end the iteration loop.

Finally, document the outcome of the iteration loop, including any changes made, remaining issues/blockers, and rationale for decisions.

### 10. Runtime verification

If the affected behavior can reasonably be exercised locally, run the application or affected service.

Follow documented repository startup procedures.

Verify that required services become ready.

Inspect logs when relevant.

For browser-visible changes, use available Playwright/browser automation tooling to verify the actual behavior.

For API/backend changes, exercise the affected API or runtime path when practical.

Do not rely solely on static code inspection when direct verification is available.

### 11. Browser verification

For web/UI behavior, use available Playwright tooling when practical.

Verify:

* affected workflow
* visible behavior
* user interaction
* application state changes
* navigation where relevant
* success/error states
* relevant console errors
* relevant network behavior

Use existing authentication state, fixtures, test data, and project conventions where available.

If existing playwright fixtures or harnesses are incomplete or insufficient consider adding the necessary fixtures or harnesses to enable comprehensive verification.

Do not consider a UI requirement complete solely because the corresponding component renders in source code.

### 12. Regression validation

Use the implementation diff to identify plausible regression areas.

Check relevant:

* callers
* shared components
* shared state
* APIs
* schemas
* persisted data
* initialization
* async behavior
* reconnect/retry behavior
* existing workflows

Run existing automated coverage for these areas when reasonable.

Do not perform arbitrary broad testing unrelated to the change.

### 13. Final validation

Before considering implementation complete:

1. inspect the complete diff
2. compare the implementation against every acceptance criterion
3. run targeted tests
4. run repository-required validation
5. run appropriate broader regression tests
6. run build/typecheck/lint where required
7. perform runtime/browser verification when appropriate
8. inspect for accidental/unrelated changes

Every acceptance criterion should be accounted for.

### 14. Clean up

Remove temporary:

* scripts
* test artifacts
* debug logging
* generated files not intended for the repository
* experimental code

Do not remove project artifacts required by the implementation.

### 15. Ticket update

When the implementation and local verification are complete, post a concise implementation update to the ticket.

Use:

## Implementation Complete

**Implementation:** <branch/MR/PR/commit when known>

### Changes

* <important change>
* <important change>

### Acceptance Criteria

* ✅ <criterion> — <implementation/verification>
* ✅ <criterion> — <implementation/verification>
* ⚠️ <criterion> — <remaining limitation if any>

### Verification

* `<command/test>` — PASS
* `<command/test>` — PASS
* Browser/runtime: <result when applicable>

### Notes

<Relevant implementation notes, limitations, or follow-up concerns.>

Do not mark the ticket as QA-approved.

Implementation verification and independent QA are separate activities.

### 16. Return a completion summary

Report:

* ticket
* implementation summary
* important files/components changed
* tests added/changed
* validation performed
* browser/runtime verification
* known limitations
* ticket update status
* branch/MR/PR when known

## Completion standard

The task is complete when:

* requirements are implemented
* acceptance criteria are addressed
* relevant automated tests pass
* repository-required checks pass
* runtime behavior has been verified when practical
* browser behavior has been verified when relevant
* no known implementation deficiency remains undocumented
* temporary artifacts are cleaned up
* the Jira implementation update has been posted

Do not claim completion merely because code has been written.
