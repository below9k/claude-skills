---
name: iot-work-ticket
description: Implement an MFB Jira ticket in FarmBot's `iot` monorepo (GitHub farmbot-monitoring-solutions/iot) end to end — readiness check, stacked-branch base, spec-first tests from the acceptance criteria, legacy parity for rewrite work, `vp` quality gates, a Codex adversarial pass, and a PR in the repo's template. Use when asked to work, implement, or pick up an MFB ticket in the iot repo (including DPS / modem-service / pivotel tickets). Runs inline so it can delegate to agents.
effort: high
---

# Work an iot ticket

Ticket:

$ARGUMENTS

This is the `iot` version of the generic `work-ticket` skill. Where this file is silent,
that skill's workflow applies. **The repo's own instructions win**: root `AGENTS.md`,
`CLAUDE.md`, `docs/adr/` (normative), and its skills under `.agents/skills/`
(`tdd`, `conventional-commit`, `raise-pr`).

## Agents

| Step | Agent | Why |
| --- | --- | --- |
| 1 | `ticket-critic` | Don't build from a ticket that isn't ready |
| 2 | `pr-stack-planner` | Pick the base when blockers are still in review |
| 3 | `legacy-source-scout` | Rewrite tickets: pin parity cases and constants from the legacy code |

Run 1 and 2 in parallel. Give each one the ticket key; relay what matters from their
reports.

## 1. Read and check readiness

- Ticket body (`responseContentFormat: html`), **all comments** (carried-forward findings
  live there), links, and the parent epic.
- Every doc rule the ticket cites (e.g. `R3`). If the Confluence doc isn't reachable,
  ask for or find a local export and name the version you used.
- `ticket-critic` verdict:
  - `READY` / `READY AFTER FIXES` → continue; list the fixes in the closeout comment.
  - `NOT READY` → stop. Show the blockers and offer `iot-ticket-write` to fix the ticket
    first. Building from a ticket that isn't ready is how "30 cases" turned into 38.
  - Proceed anyway only if the user says so, and record that in the PR notes.

## 2. Branch

- Base from `pr-stack-planner`. Name: `<type>/mfb-<n>-<slug>` (e.g.
  `feat/mfb-1607-dispatcher`). Work in a worktree if other work is in flight.
- If repo instructions require Beads (`bd`), follow `docs/agents/issue-tracker.md`: the bd
  issue carries the Jira key as an external reference. Jira stays the org-facing record.

## 3. Specs first

Turn each acceptance criterion into a failing test **before** implementing:

- Rules ops or product would quote (planner decisions, FRED rules, public HTTP contracts)
  → Gherkin `.feature` with `@amiceli/vitest-cucumber` (ADR 0003). Look at an existing
  `bdd/features/` folder in a sibling app or package for layout.
- Everything else → Vitest unit or integration tests next to the code, using the package's
  existing style. Inject clocks, `fetch` and gate functions (`isGateOpenFn`) rather than
  mocking modules.
- **Rewrite parity:** have `legacy-source-scout` inventory the legacy tests for the
  component and port them as scenarios. If the count differs from the ticket, the code
  wins; note the difference for the closeout.
- Confirm the new tests fail for the right reason.

## 4. Implement

- Follow the ticket's `Pattern:` path; read it before writing anything new.
- Keep to the ticket's scope. When something belongs to another ticket, write it down for
  the closeout ("Deferred to MFB-…") and move on.
- Platform rules that bite: queues only in `infra/data-plane` via `iotQueueName()`
  (ADR 0006); DB through `createDb` / `authTokenProvider` (ADR 0005/0015); kill-switch
  readers fail closed; no secrets in logs.

## 5. Gates

Run whichever apply, narrowest first, and record each with its result:

```sh
vp run --filter @iot/<pkg> test        # while iterating
vp check                               # format + lint + typecheck
vp test
vp run -r build                        # if build/CDK surfaces changed
vp run fallow
vp run contracts:check && vp run contracts:lint   # if spec/ or @iot/contracts changed
vp run --filter "@iot/db" db:ci        # if packages/db migrations changed
```

## 6. Adversarial pass

Run Codex read-only against the branch diff, using the invocation in the
`gitlab-code-review` skill's "Adversarial Review (Codex CLI)" section. Give it the ticket
AC, the diff command (`git diff <base>...HEAD`), and focus areas that fit the change
(concurrency, secret leakage, URL policy, idempotency, partial failure). Verify each
finding yourself, then sort it:

- **Fixed** — fix it now, with a test.
- **Deferred to <KEY>** — belongs to another ticket in the epic.
- **Carried to <KEY>** — changes how a *later* ticket must be built. Goes to the closeout.
- **Confirmed fine** — checked and not a problem; say why in one line.

If Codex isn't available, say so in the PR. Don't skip silently.

## 7. Commit and PR

- Conventional commits, scope = package (`feat(pivotel-client): …`). No ticket key in the
  subject.
- Push and open the PR only with the user's go-ahead. Use the repo's `raise-pr` skill
  (fills `.github/pull_request_template.md`). On top of the template:
  - **Stacking:** `Stacked on #<n>` when the base isn't `main`.
  - **Refs** line with the ticket and epic keys.
  - The four-bucket adversarial result from step 6 under **Test plan**.
  - **Notes for reviewers → Intentionally deferred** mirrors the ticket's Out of scope.

## 8. Close out

Hand off to `iot-ticket-closeout`: verify the AC against the PR, post the Jira comment,
tick the boxes, and carry findings to downstream tickets (all with approval).
