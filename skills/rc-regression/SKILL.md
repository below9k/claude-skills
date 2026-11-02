---
name: rc-regression
description: >
  Use this skill when asked to regression-test, QA, or validate a Savi
  release candidate, such as release/v3.5.8-rc1. Defaults to ticket-by-ticket
  review of every Jira ticket in the RC (from the git range against the last
  shipped release, cross-checked with Jira), plus risk-based regression
  around them; can instead run a general regression of core Savi
  functionality, or both. Deploys the RC's package-sled artifact to the test
  server and tests the running system — live interactive browser testing
  with Playwright MCP, SSH/system checks, and code tests where useful. Builds
  simulated devices, extra IPs, HTTP capture servers, or other harnesses when
  a behavior cannot be exercised otherwise, and reports each one so it can
  become future automated test coverage.
context: fork
agent: rc-regression
effort: high
---

# Release Candidate Regression

Regression-test the Savi release candidate identified in the user's request.

The user's request is:

$ARGUMENTS

## Purpose

Decide whether a release candidate is fit to ship, based on the running
system rather than on code tests or pipeline status. Every result must come
from something that was actually exercised.

This is a QA pass: do not fix defects unless the user explicitly asks.

## Inputs and Modes

The user may provide:

- The RC branch, e.g. `release/v3.5.8-rc1` (required; ask if missing).
- A mode:
  - **By ticket** (default): test every ticket in the RC, plus the
    regression surface around each.
  - **General**: a broad regression of core Savi functionality,
    independent of specific tickets.
  - **Both**: tickets first, then the general pass over areas not already
    covered.
- A subset of tickets to limit the ticket pass.
- An explicit baseline release to diff against.
- An alternate test server.

Never invent tickets, versions, branches, or artifacts.

## Shared Procedures

This skill reuses the deploy and testing procedures in
`~/.claude/skills/qa-ticket/SKILL.md`. Read that file before starting. Apply
these sections from it exactly, substituting the RC artifact for the MR
artifact:

- **Default Test Server** and **Server Safety**
- **Download Artifact** through **Verify Installed Build** (download,
  upload to `/tmp`, install, `reboot -f`, recovery, version check)
- **Live Interactive UI Testing**, including Verify Effects Beyond the UI,
  Positive and Negative Paths, Preserve the Test Environment, and Evidence
- **System Testing**
- **Investigate Failures** and **Retesting**
- **Test Harnesses and Simulations**
- **Result Definitions** and **Safety Rules**

Where this skill and `qa-ticket` differ, this skill wins.

---

# Phase 1 — Identify the RC and Its Baseline

Work in a Savi checkout (`~/develop/savi*`) with the canonical remote
fetched. Remotes vary by checkout (`upstream`, `base`, …), so inspect
`git remote -v` rather than assuming.

1. Resolve the RC branch on the canonical remote, e.g.
   `<remote>/release/v3.5.8-rc1`, and record its HEAD commit.
2. Determine the **release line**: `release/v3.5.8-rc1` belongs to
   `release/v3.5.8`.
3. Determine the **baseline**: the most recent *shipped* release before this
   one. That is the highest `release/vX.Y.Z`, `release/vX.Y.Z.N`, or
   `release/vX.Y.Z-hfN` branch below this version. Skip `-rcN` and ad-hoc
   branches. Versions are not always contiguous; for example, 3.5.8 follows
   3.5.6.1, and there is no 3.5.7.
4. For `-rc2` and later, also note the previous RC. Tickets changed since
   the previous RC get priority, but all RC tickets stay in scope.
5. If the user named a baseline, use it. If several are plausible, state
   which one you chose and why.

Record:

    RC branch:
    RC commit:
    Release line:
    Baseline:
    Previous RC:

---

# Phase 2 — Build the Ticket Inventory (by-ticket mode)

Build the inventory from two sources, then reconcile them:

1. **Git:** `git log --no-merges --format='%h %s' <baseline>..<rc>`, plus
   merge commit subjects and MR titles. Extract keys such as `SN-`, `DSP-`,
   `GAL-`, and `SLED-`. Use GitLab to map each merged MR to its ticket.
2. **Jira:** query tickets whose fix version matches the release (e.g.
   `3.5.8`), using the project's actual fix-version naming.

Reconcile the two:

- **In both:** in scope.
- **In git only:** in scope. Note that the fix version is missing.
- **In Jira only:** flag as **possibly missing from the RC**. It may not have
  been merged, or it may be on a different release branch. Verify it before
  testing; if the change is absent, that is a finding in itself.

For each ticket, retrieve it with the Jira MCP. Capture the acceptance
criteria (sometimes in `customfield_10160`), the commits, the MR, and the
changed packages.

If a ticket lacks usable acceptance criteria, derive them by following
`~/.claude/skills/qa-acceptance-criteria/SKILL.md`. Do not post those
criteria to Jira during an RC run unless the user asked. Mark them
QA-derived in the report.

## Prioritize

Order the tickets by risk, using:
- blast radius (shared modules, the mesh, auth, persistence, upgrade paths)
- user visibility
- device/driver impact
- the size of the change
- whether the ticket changed since the previous RC

Test the highest-risk tickets first so an early blocker surfaces early.

Present the inventory as a table before you start testing:

    | Ticket | Summary | MR | Packages | Risk | Test approach |

---

# Phase 3 — Build the General Regression Inventory (general mode)

Derive the inventory of core functionality from the system itself, not
from memory:

- The UI navigation of the running app: every top-level area and its
  primary workflows.
- The repo's `CLAUDE.md` and `architecture.md`.
- `savi.yaml`, which lists every service and driver process that should be
  running.
- The packages changed between the baseline and the RC, which get extra
  weight.

At minimum, cover these areas when present:
- Every expected process is running and healthy after the upgrade and
  reboot.
- Login and authentication.
- Navigation to every top-level area.
- Device and driver discovery, control, status feedback, and reconnect.
- Configuration save, reload and persistence.
- Scheduling and events.
- Media, streams and displays.
- The API and browser WebSocket bridge.
- Upgrade from the baseline: data and settings survive the upgrade
  (see Phase 4).
- Logs free of new errors after the tests.

Write the inventory to the run directory (Phase 7) as `general-checklist.md`.
Report its path so later RCs can reuse and extend it.

---

# Phase 4 — Select, Deploy, and Verify the RC Build

Select the artifact from the latest successful pipeline on the RC branch.
Use the `package-sled` job's `artifacts/package/*` for the sled; other
components have their own jobs, e.g. `build-dsp`. Confirm that:
- the pipeline commit equals the RC HEAD
- the artifact has not expired
- the version string matches the RC

Then follow the qa-ticket deploy sections (download through Verify Installed
Build).

**Upgrade path:** when practical, and when the user has not said otherwise,
install the **baseline** build first. Record representative configuration
and data, then upgrade to the RC and verify that configuration and data
survived. Upgrades are where regressions hide. If this is not practical,
test a fresh RC install and list upgrade testing as not covered.

---

# Phase 5 — Execute Testing

## Ticket pass

For each ticket, in priority order, apply the qa-ticket testing sections:
- For user-facing criteria, do live interactive browser testing (a rendered
  page is not a pass).
- Verify backend, device and system effects over SSH, logs and APIs.
- Test positive and negative paths.
- Run targeted regression checks immediately around the ticket's change.

Record one result per criterion: PASS, FAIL, or BLOCKED.

## General pass

Work through the general checklist using real UI workflows, not page loads.
Mark each item PASS, FAIL, BLOCKED, or NOT COVERED.

## Code tests (supplementary)

Run automated specs where they add signal, particularly for changed
packages. On a Savi checkout, follow the runner rules in
`~/.claude/agents/spec-runner.md` (line detection, no improvised runners,
failure triage). Code tests supplement behavioral testing; they never
replace it.

## Blockers

If a blocker stops a whole area (e.g. the install fails or a core service
crashes), record it, investigate it enough to classify it, and continue
with the areas that are still testable. Do not abandon the run.

---

# Phase 6 — Simulations and Harnesses

When a behavior cannot be exercised with the real hardware and network
available, build what is needed rather than marking it BLOCKED. Examples:
- simulated devices that speak a driver's protocol
- additional IP addresses so several simulated devices can coexist
- HTTP, TCP or UDP servers that capture outbound requests from the sled
- fake cloud or third-party endpoints
- scripted mesh clients using `interfaceFactory`
- fault injection: dropped connections, slow responses, error codes

Follow the qa-ticket **Test Harnesses and Simulations** rules. In
particular:
- Reuse existing fakes first. Search the repo, e.g.
  `lerna/packages/webos3-simulator`, `driver-*/test-tools/` (such as
  `fake-everpass`), `core/interface-factory-mock`, `core/utils/mock`, and
  `webos3-app/src/libs/fake_scap`.
- Keep harnesses outside the repo.
- Never claim an IP address or port that another host on the network might
  be using.
- Restore every network and configuration change afterwards.
- Label every result that depended on a simulation as simulated.

---

# Phase 7 — Run Directory and Report

Keep all run output in `~/qa-harness/<rc-name>/`, e.g.
`~/qa-harness/release-v3.5.8-rc1/`. That includes:
- the report
- `general-checklist.md`
- the harness source (under `harness/`)
- evidence (screenshots, log excerpts)

Never write any of it into a Savi checkout.

Write the report to `report.md` there, using this structure:

    # RC Regression — <rc branch>

    ## Summary
    Recommendation: GO | GO WITH NOTES | NO-GO | BLOCKED
    Mode: by ticket | general | both
    Tickets: <passed>/<failed>/<blocked> of <total>
    General checks: <passed>/<failed>/<blocked>/<not covered> of <total>
    Release-blocking issues: <n>

    ## Build
    RC branch / commit:
    Baseline:
    Pipeline / job:
    Artifact:
    Installed version:
    Server:
    Upgrade path tested: yes (from <baseline>) | no, because ...

    ## Inventory Reconciliation
    Tickets in git and Jira / git only / Jira only (possibly missing)

    ## Ticket Results
    | Ticket | Result | Criteria (P/F/B) | Simulated? | Notes |
    Then, per ticket, the qa-ticket acceptance-criteria and regression
    blocks, with the interactions performed.

    ## General Regression
    [PASS|FAIL|BLOCKED|NOT COVERED] <area> — <what was exercised>

    ## Issues Found
    Per issue: severity, release-blocking yes/no, ticket or area, steps to
    reproduce, expected, actual, evidence, classification (regression vs
    pre-existing vs environment, etc.), likely code area.

    ## Simulations and Harnesses
    (per the qa-ticket harness report format)

    ## Coverage
    Tested / not tested (and why)

**Recommendation rules:**
- **NO-GO:** any unresolved release-blocking defect or regression.
- **BLOCKED:** a required area could not be tested.
- **GO WITH NOTES:** only minor or pre-existing issues were found.
- **GO:** everything in scope passed.

Never give GO while a required ticket or area is untested.

## Jira

Do not post to Jira during an RC run unless the user asks. RC runs touch
many tickets, and comments on each would be noise.

When asked, post one short RC-labelled result comment to each ticket
tested. It should cover:
- the RC and build
- per-criterion results
- defects, with reproduction steps
- a pointer to the full report

Do not transition statuses unless told to.

## Return to the User

Return:
- the recommendation
- the counts
- the release-blocking issues
- a list of the harnesses that were built
- what was not covered
- the path to `report.md`

The report is the detailed record.
