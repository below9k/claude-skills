# iot ticket specifics

Read on top of `~/.claude/skills/ticket-write/ticket-standard.md` by `ticket-write`,
`backlog-sync`, `ticket-critic`, `ticket-closeout`, `epic-loop` and `legacy-source-scout`.

## Anatomy additions

- Summary: `<ID> <outcome noun phrase>`, e.g. `P3.7 Dispatcher loop`. `P#.#` IDs come from
  the design doc's backlog (phase.item) and are replaced by `MFB-` keys after creation.
- Footer: `ID: P3.7 · Slice: 3 · Size: L · Blocked by: MFB-1597, MFB-1599 · Source: <doc anchor>`.
- Rules ops or product would quote belong in a `.feature` file (ADR 0003); the ticket names
  which one.
- Tasks (P0.x, P8.x) use the Task anatomy, not Story.

## Design docs

- Confluence on `farmbot-au.atlassian.net`, which may not be granted to the MCP. Fall back
  to a local export and say which version you used. For DPS the doc is the epic page
  "Epic: Rewrite M3S as the Device Provisioning System" (the TSD); it exists only as a PDF
  export, so extract its text yourself (`pdftotext`). Exports drop the collapsed ticket
  headings: match bodies by order within a phase.
- Doc references: `R#` (rules), `D#` (decisions), `G#` (goals). Never cite one bare.

## Labels

The repo's triage vocabulary is in `docs/agents/triage-labels.md`. Proposed lifecycle
(standard §3.9a): `needs-triage` → `needs-info` / `ready-for-agent` / `ready-for-human`.
DPS tickets all currently carry `triage`.

## Legacy source (rewrite work)

- M3S: `farmbot-monitoring-solutions/m3s`, .NET 10 / C#. Read its `CLAUDE.md` and `docs/`
  first. Use a local clone if there is one, else
  `gh api repos/farmbot-monitoring-solutions/m3s/contents/<path> --jq .content | base64 -d`.
- Skip `bin/` and `obj/`. Tests live in `*Tests`, `*IntegrationTest` and `LocalE2E`
  projects; count cases as `[Fact]` + `[Theory]` × `[InlineData]` rows.
- Per-environment config overrides: `appsettings.*.json` and CDK.

## PR and Jira lookups

- PRs touching a ticket: `gh pr list -R farmbot-monitoring-solutions/iot --search <key>`.
- Epic PRs: `gh pr list --search "head:feat/mfb- OR head:docs/mfb-" --state all`.
- Their "Deferred to" and "Carried to" lines are requirements for other tickets.
