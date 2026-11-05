---
name: savi
match:
  remotes: ["savicontrols/savi"]
  markers: ["lerna/packages"]
---

# Savi

The Savi sled monorepo: a long-running Node process set (sled-daemon services, drivers, a
Meteor app) talking over a hemera/NATS service mesh to flaky hardware. GitLab
`savicontrols/savi`. Local checkouts live in `~/develop/savi*`.

## Topic files

| File | Holds | Read by |
| --- | --- | --- |
| `architecture.md` | the mesh house rules, path rules, the seven blast-radius planes, async hotspots | `conventions-reviewer`, `impact-tracer`, `async-bug-hunter`, `work-ticket` |
| `testing.md` | runner paths per line, spec house style, the DSP toolchain state | `spec-runner`, `spec-author`, `work-ticket` |
| `runtime.md` | the dev sled: pm2, bring-up, browser and mesh flows, log correlation | `work-ticket`, `review-follow-up` |
| `qa-env.md` | the test server, artifacts, install/reboot, existing fakes, the general regression list | `qa-ticket`, `release-regression` |
| `release.md` | release lines, version bumps, tags, the finalization checklist, mergeback hotspots, migrations | `release`, `mergeback`, `mergeback-auditor`, `release-ticket-triage`, `release-regression` |

## Two lines: detect which one you are on

Run at the repo root:

```sh
ls -d lerna/packages savi.yaml CLAUDE.md 2>&1; git branch --show-current
```

- `savi.yaml` present → **main line**. The repo's `CLAUDE.md` and `architecture.md` exist and
  win over this profile wherever they differ.
- No `savi.yaml` → **DSP line** (`release/dsp/*` and branches cut from it). There is no
  `CLAUDE.md`, `architecture.md` or `PACKAGES.md`; the rules in `architecture.md` here are
  the only written copy. Don't cite repo docs that don't exist on this line.

Name the line in your output. The differences that matter:

| | main line | DSP line (`release/dsp/v1.1.3`) |
| --- | --- | --- |
| `CLAUDE.md` / `architecture.md` | present (no `PACKAGES.md`) | **absent** |
| Process list | `savi.yaml` + `savi-prod.yaml` (both must change together) | **`dev.app.config.js`**: `daemonSrvs` + `driverProcesses` |
| pm2 | `./cli-bazel/pm2` (regenerates the ecosystem from yaml) | **`./cli-bazel/pm2.sh`** (passthrough to the vendored pm2) |
| Bring-up | `./cli-bazel/pm2 start` (check `CLAUDE.md`) | **`./cli-bazel/start-savi.sh`** (does `pm2 delete all` first) |
| Test macro | `savi_node_test` in some packages; specs mixed | **`savi_mocha_test`** → `//<pkg>:unit_test` (19 packages) |
| Spec style | match the package's existing specs | **mocha + chai + sinon** only |
| Fast local runner | `.ai-dev/tools/jest-shim/` | none |
| WebSocket client | two packages | none: the `websocket` sled-daemon service is the bridge |

## Tracker

- Jira cloudId `729cdf11-9f14-49de-9b13-464494ee2751` (Confluence on
  `oliveriq.atlassian.net`).
- Project keys: `SN` (most work), `DSP`, `GAL`, `CLD`, `SLED`. Check commit subjects on a
  branch for others.
- Acceptance criteria are sometimes in `customfield_10160` instead of the description.
- fixVersions are quarter-named (`2026-Q3.2`, `2026-Q4.1`), not `3.5.x`. Never derive one
  from a branch name.

## Forge and remotes

- GitLab, project `savicontrols/savi` (`savicontrols%2Fsavi` in API paths). Use the GitLab
  MCP; `glab` if configured.
- **Fork workflow.** Push feature branches to the user's fork; open the MR from the fork into
  the canonical repo. Remote names vary by checkout: usually `origin` = fork and `upstream`
  = canonical, but `~/develop/savi` uses `1fork` = fork and `base` = canonical. Check
  `git remote -v`.
- **Identity is functional.** CI only triggers for commits, pushes and MRs made under the
  user's own GitLab identity. Never commit, push or open an MR as another account; if the MR
  doesn't show the user as author, investigate before calling it done.
- Top-level MR notes can fail on the MCP token's scopes; reply inside existing threads
  (`create_merge_request_discussion_note`). The token cannot read some MR notes (403); say so
  rather than guessing.

## Branches

- Feature branches fork from a **release branch**, not `main`/`master`/`develop`:
  `release/v#.#.#` (sled software) or `release/<project>/v#.#.#` (e.g. `release/dsp/v1.1.3`,
  `release/eed/v1.0.1`). Pick the release the work belongs to; ask when several are
  plausible. Diff and review against that base (`git merge-base`).
- Name: `PRJ-####/PRJ-###-<kebab-case-description>` (parent ticket / work item / summary),
  e.g. `SN-1234/SN-567-fix-audio-subscription-cleanup`. Never invent ticket numbers.
- Exception: mergebacks use `SN-mergeback-<src>-to-<tgt>` (see `release.md`).
- Never push to a `release/*` branch or `develop` directly, except when cutting a release
  branch (see `release.md`).

## Commits and PRs

- Ticket-prefixed subjects: `SN-567 Fix audio subscription cleanup`. MR title the same.
- MR description: what changed, why, implementation notes, testing performed, limitations.
- After opening an MR, link it on the Jira ticket (remote link, or a short comment).
- Under review: add commits on top; don't amend, squash, rebase or force-push.

## Validation

See `testing.md`. Short version: main line uses
`./cli-bazel/bazel test //<pkg>:<target>` (target from `BUILD.bazel`); the DSP line has no
working fast path (observed 2026-09-09) and `spec-runner` must say "could not run" rather
than improvise. Lint a bazel package with `./cli-bazel/lint //<pkg>`.

## Agents

| Agent | Run it when |
| --- | --- |
| `impact-tracer` | before changing a shared interface, IoC key, service, or process registration |
| `conventions-reviewer` + `async-bug-hunter` (in parallel) | on any diff under `lerna/packages/` |
| `spec-author` / `spec-runner` | new behaviour or a bug fix needs a spec; after edits |
| `mergeback-auditor` | before pushing a mergeback, or reviewing someone else's |
| `release-ticket-triage` | while finalizing a release |
