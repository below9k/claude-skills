---
name: iot
match:
  remotes: ["farmbot-monitoring-solutions/iot"]
  markers: ["pnpm-workspace.yaml", "AGENTS.md", "CONTEXT.md"]
---

# FarmBot iot

FarmBot's `iot` TypeScript monorepo (`apps/*`, `packages/*`, `infra/*`), GitHub
`farmbot-monitoring-solutions/iot`. Current large epic: MFB-1587, the DPS rewrite (M3S →
Device Provisioning System). Loop state for it lives in `~/.claude/state/epic-loop/MFB-1587/`.

## Topic files

| File | Holds | Read by |
| --- | --- | --- |
| `tickets.md` | MFB ticket anatomy specifics, design-doc sites, labels, the legacy M3S repo | `ticket-write`, `backlog-sync`, `ticket-critic`, `ticket-closeout`, `epic-loop`, `legacy-source-scout` |

## Repo instructions (these win)

Root `AGENTS.md` and `CLAUDE.md`; `docs/adr/` (normative); `CONTEXT.md` (glossary: write
tickets and specs in its words); `docs/agents/triage-labels.md`;
`docs/agents/issue-tracker.md` (if Beads `bd` is required, the bd issue carries the Jira key
as an external reference; Jira stays the org-facing record); repo skills under
`.agents/skills/` (`tdd`, `conventional-commit`, `raise-pr`).

## Tracker

- Jira `farmbot.atlassian.net`, project `MFB` (call `getAccessibleAtlassianResources` once
  for the cloudId). Keys `MFB-\d+`.
- Write and read bodies as **HTML** (`contentFormat: html`, `responseContentFormat: html`).
  Markdown task lists become literal `\[ \]` bullets.
- Every Jira mutation needs explicit approval in the conversation (rule in
  `docs/agents/triage-labels.md`), unless a standing authority in the epic's loop state
  covers it.
- Statuses seen: BACKLOG, Selected for Development, In Progress, PR REVIEW, Done.

## Forge and remotes

- GitHub, `gh` CLI. **No fork**: branches push to `origin` (the canonical repo).
- SSH push fails. Push with
  `git -c credential.helper='!gh auth git-credential' push https://github.com/farmbot-monitoring-solutions/iot.git HEAD:<branch>`.
  Because that push goes to a URL, it doesn't update `origin/<branch>`, and a bare
  `--force-with-lease` is always rejected (`stale info`): pin it,
  `--force-with-lease=<branch>:<old head SHA>`. Never plain `--force`.
- Work ships as **stacked PRs**, one ticket per PR (e.g. `#449 → #446 → #444 → main`). Use
  `pr-stack-planner` to pick a base and to restack.

## Branches

- Base: `main`, or the branch of an unmerged blocker (from `pr-stack-planner`).
- Name: `<type>/mfb-<n>-<slug>` (e.g. `feat/mfb-1607-dispatcher`, `docs/mfb-1587-…`). Use a
  worktree when other work is in flight.

## Commits and PRs

- Conventional commits, scope = package: `feat(pivotel-client): …`. **No ticket key in the
  subject.**
- Open PRs with the repo's `raise-pr` skill (fills `.github/pull_request_template.md`), only
  with the user's go-ahead. Add on top of the template:
  - `Stacked on #<n>` when the base isn't `main`;
  - a **Refs** line with the ticket and epic keys;
  - the adversarial-review buckets (Fixed / Deferred to / Carried to / Confirmed fine) under
    **Test plan**;
  - **Notes for reviewers → Intentionally deferred**, mirroring the ticket's Out of scope.
- Reviews come in waves from a teammate (`🟡 Needs triage` items); each blocks the merge
  until answered.

## Validation

`vp` is a zsh function: call it as `zsh -lic 'vp …'`. A fresh worktree needs `vp install`
and `vp run -r build` before `vp check` or contracts. Narrowest first:

```sh
vp run --filter @iot/<pkg> test        # while iterating
vp check                               # format + lint + typecheck
vp test
vp run -r build                        # if build/CDK surfaces changed
vp run fallow
vp run contracts:check && vp run contracts:lint   # if spec/ or @iot/contracts changed
vp run --filter "@iot/db" db:ci        # if packages/db migrations changed
```

- `contracts:check` compares against **committed** files.
- A new Fargate app must be added to the census lists:
  `packages/db/test/deploy-wiring.test.ts`, `packages/contracts/test/openapi-bundle.test.ts`,
  and the `server.js` guards in `.github/workflows/{ci,deploy-environment}.yml`.

### Specs

- Rules ops or product would quote (planner decisions, FRED rules, public HTTP contracts) →
  Gherkin `.feature` with `@amiceli/vitest-cucumber` (ADR 0003). Copy the layout of an
  existing `bdd/features/` folder.
- Everything else → Vitest next to the code, in the package's style. Inject clocks, `fetch`
  and gate functions (`isGateOpenFn`) rather than mocking modules.

## Architecture rules (for `conventions-reviewer`)

The ADRs are normative; these are the ones that bite most:

- Queues only in `infra/data-plane`, named via `iotQueueName()` (ADR 0006).
- DB access through `createDb` / `authTokenProvider` (ADR 0005, ADR 0015).
- Kill-switch readers fail closed.
- No secrets in logs.

## Agents

| Agent | Run it when |
| --- | --- |
| `ticket-critic` | before starting a ticket, on every draft, on an epic's backlog |
| `pr-stack-planner` | before starting a ticket whose blockers are in review; after any stacked PR merges |
| `legacy-source-scout` | rewrite tickets (M3S → DPS): pin parity cases and constants |
| `conventions-reviewer` | on any diff (checks the ADRs) |

Adversarial focus areas that fit this repo: concurrency, secret leakage, URL policy,
idempotency, partial failure.
