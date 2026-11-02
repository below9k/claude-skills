---
name: work-ticket
description: Implement an engineering ticket such as SN-#### from start to finish. Use when asked to work, implement, fix, develop, or complete a Jira ticket. Retrieves the ticket, reads repository Claude instructions, identifies the associated code and branch, writes down an end-to-end tracer bullet (golden path) for the requested behavior before editing, implements along that path, verifies it, self-reviews every boundary on the path, iterates on failures, and reports completion. For addressing review comments on an existing MR, use review-follow-up instead.
effort: high
---

# Work Ticket

Work the engineering ticket identified in:

$ARGUMENTS

## Savi repositories: delegate to the specialist agents

In a Savi repository (`lerna/packages/` at the root), use the Savi subagents at the steps
below. They detect the DSP vs main line themselves. Give each one the ticket key, the base
branch, and the specific files or interfaces in question; relay what matters from their
reports rather than re-deriving it.

| Step | Agent | When |
| --- | --- | --- |
| 4–5. Trace impact | `impact-tracer` | While building the tracer bullet, when it crosses a shared interface, IoC key, service, or process registration |
| 5. Plan | `slice-planner` | Non-trivial tickets, after the tracer bullet is written and before implementing |
| 7. Tests | `spec-author` | New behavior or a bug fix needs a spec (and a red-before-fix check) |
| 8, 13. Validation | `spec-runner` | After meaningful edits, and in final validation |
| 13. Review | `savi-invariant-reviewer` + `async-bug-hunter`, in parallel | Final validation, on the complete diff |
| 10. Runtime | `sled-e2e` | When wiring (hemera topics, IoC order, pm2 startup) is in question |

Outside Savi repositories, do these steps yourself.

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

If the repository's documented workflow requires a ticket branch, follow it. For GitLab
repositories (`~/develop/savi*`), use the `gitlab-push` skill's base-branch detection and
branch naming (`PRJ-####/PRJ-###-<kebab-case-description>`, cut from the detected
`release/v#.#.#` or `release/<project>/v#.#.#` branch).

Do not commit or push unless the user asks. When they do, use the `gitlab-push` skill.

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

Follow the requested behavior from where it starts (a user action, a device event, a
schedule, a startup hook) to where its result is observable, reading each hop in the code.
Step 5 writes that path down; collect it here rather than only the files that look relevant.

Do not speculate about code you have not inspected.

### 5. Define the tracer bullet, scope, and plan

Before modifying code, write down the tracer bullet (the golden path): the smallest
complete path through the existing system that carries the requested behavior from its
entry point to its final observable result. Do this before deciding which files to modify.
It exists to prevent changes that are locally correct but fail somewhere else on the path,
which is what code review most often catches.

#### Write the tracer bullet

Use concrete components, interfaces, handlers, processes, events, and files where they are
known. It is a behavioral map, not pseudocode, so write no implementation code in this step.

For example, in a Savi repository (names are illustrative):

    User picks a region state in the layout editor
        ↓
    LayoutRegionPanel → canvas interface caller method (@savi/sled-websocket-client)
        ↓
    sled-websocket-server bridge → NATS
        ↓ [RISK: driver process restarting; dependsOn gating]
    canvas display driver handler (registered via interfaceFactory)
        ↓
    driver → device command
        ↓ [RISK: device disconnects mid-command]
    device confirms → driver publishes new state
        ↓
    UI subscription
        ↓
    LayoutRegionPanel shows the confirmed state

The real path may be shorter or longer. Do not invent layers to make it look complete, and
when no meaningful multi-component path exists, use the simplest accurate one.

#### Define success and failure

State the observable result that proves the path works, and map it to the acceptance
criteria:

    Success: choosing a state sends it to the device, the confirmed state propagates
    back, and the editor shows the confirmed value.

When relevant, state the important failure behavior too:

    Failure: if the device rejects or never receives the command, the editor must not
    keep showing an unconfirmed value.

#### Validate each transition

Walk every transition against the existing code before implementing. For each meaningful
boundary, determine:

* what crosses it, and the data shape or contract
* who owns each side
* whether it is synchronous or asynchronous
* error behavior on both sides
* who owns cleanup (listeners, subscriptions, timers, sockets)
* whether it can run more than once, concurrently, or deliver a stale or out-of-order result
* other consumers of the same contract
* existing test coverage

Mark plausible risks on the tracer bullet as `[RISK: ...]`. Pay most attention to the
boundaries where defects concentrate: UI → state, caller → handler, service → device,
event → subscriber, persistence → cache, request → async response, connect → disconnect,
create → cleanup, retry → duplicate, startup → shutdown. In Savi, also check transport
choice (interfaceFactory vs localIntfFactory, WebSocket vs NATS), `dependsOn` restart
ordering, and the process the handler is registered in. Stick to risks the architecture
supports, not hypothetical ones.

#### Turn the path into the plan

Write a concise checklist organized around behavior, not around files. Each item maps to a
part of the tracer bullet:

    [ ] editor initiates the state change
    [ ] interface accepts and validates the input
    [ ] driver handler performs the operation
    [ ] device receives the command
    [ ] confirmed state propagates back
    [ ] editor shows the confirmed state
    [ ] reject/disconnect path behaves correctly
    [ ] subscriptions and timers are cleaned up
    [ ] existing consumers of the changed contract still work
    [ ] tests prove the important parts of the path

Also identify:

* code that needs to change
* tests that need to change or be added
* likely regression areas
* validation required before completion

Prefer the smallest implementation that completely satisfies the ticket.

Do not add unrelated refactors or improvements.

### 6. Implement incrementally

Make focused changes in logical increments.

Work through the tracer bullet from start to end, checking off its checklist items as each
one is done. After modifying a layer, check its contract with the next layer before moving
on. A component that compiles or passes an isolated test does not prove the next component
receives what it expects.

When implementation shows the tracer bullet was wrong or incomplete, stop and update the
tracer bullet and checklist before continuing. It is a living model of the change, not a
fixed first guess.

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

Map tests to the tracer bullet. For each meaningful test, be able to answer: *which part of
the tracer bullet does this test prove?* Prioritize:

1. the primary success path
2. important boundary transitions
3. plausible failure paths
4. regression-sensitive existing behavior

Every box on the path does not need its own test when higher-level coverage already proves
it. If an important boundary has no meaningful coverage and a regression there would matter,
add coverage using the repository's conventions. Do not add tests that raise coverage
without proving behavior.

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

Also use the tracer bullet: for each changed step or contract, trace outward to its other
existing consumers and the workflows they serve. A regression can land outside the files the
ticket changed, especially when the change touches a shared service, interface, event,
subscription, state object, schema, device command, initialization path, or cleanup.
Validate important existing behavior on both sides of every changed boundary.

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

#### Tracer bullet self-review

Before the specialist reviewers run, review your own change against the tracer bullet as if
it were another developer's merge request. Look at every boundary the path crosses, not only
the files you meant to change. For each step ask:

* Does this step actually satisfy its part of the requirement?
* Does the next consumer receive exactly what it expects?
* Did changing this contract affect another consumer?
* Is an error handled on one side but ignored on the other?
* Can this run twice, or concurrently? Can an old response overwrite newer state?
* What happens if execution stops halfway through?
* What happens on disconnect, reconnect, process restart, or page reload?
* Are listeners, subscriptions, timers, sockets, and processes cleaned up?
* Do the tests prove the behavior, or only execute the code?

Then write out the completed path with its status:

    ✓ editor selection
        ↓
    ✓ interface caller → WebSocket → NATS
        ↓
    ✓ driver handler
        ↓
    ✓ device command and confirmation
        ↓
    ✗ UI subscription (not verified: no device on the dev sled)

Mark a step ✓ only once it is implemented and reasonably validated. If the review exposes a
defect or a missing path, go back to that step, fix it, update tests where appropriate,
rerun the affected validation, and repeat that portion of this review. Move on to the
specialist reviewers only once the tracer bullet holds. They do not replace this review:
it asks whether the feature works end to end, and they look independently for broken
invariants and async defects.

#### Checklist

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
* the final tracer bullet with each step's status, and the risk paths verified
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
* the tracer bullet has been walked end to end, with any unverified steps called out
* relevant automated tests pass
* repository-required checks pass
* runtime behavior has been verified when practical
* browser behavior has been verified when relevant
* no known implementation deficiency remains undocumented
* temporary artifacts are cleaned up
* the Jira implementation update has been posted

Do not claim completion merely because code has been written.
