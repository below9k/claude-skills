# Savi subagents

These agents are Savi-specific and branch-aware. Each one first detects which Savi line it is in:
`savi.yaml` at the repo root means the **main line** (the repo's `CLAUDE.md` / `architecture.md`
win), and its absence means the **DSP line** (`release/dsp/*`), whose rules are inlined into the
agents. Outside a Savi repo (no `lerna/packages/`) they stop and say so.

Agent definitions for the Claude Code `Agent` tool. Each file is one agent; the frontmatter
`description` is what the model matches against when deciding to delegate, so it is written
to say *when to use it*, not just what it is.

Invoke by name, e.g. "use impact-tracer on the driver-command-router interface".

| Agent | Use it when |
| --- | --- |
| `slice-planner` | Before starting non-trivial work. Turns a ticket into a tracker of verifiable slices. Read-only. |
| `impact-tracer` | Before editing a shared interface, key, or service. Maps the blast radius across seven planes. Read-only. |
| `savi-invariant-reviewer` | On any diff under `lerna/packages/`. Checks the mesh house rules — transport, data access, `dependsOn`, process registration. |
| `async-bug-hunter` | On driver, DSP, proxy, queue, or transport changes. Hunts promise-lifetime, timer-leak, reconnect, retry, partial-failure bugs. |
| `spec-runner` | After edits, before a commit. Runs the affected `:unit_test` targets and triages every failure. |
| `spec-author` | When a change lands untested, or a bug needs a regression spec pinned. Writes mocha/chai specs. |
| `sled-e2e` | When a change needs proving on a running dev sled. Asks before touching pm2. |

## How they compose

```
slice-planner  →  impact-tracer (on the risky slice)  →  implement
               →  spec-author  →  spec-runner
               →  savi-invariant-reviewer + async-bug-hunter (in parallel)
               →  sled-e2e (only if wiring is in question)
```

The two reviewers are deliberately split and can run at the same time.
`savi-invariant-reviewer` answers "does this obey our house rules"; `async-bug-hunter`
answers "is this correct". Neither tries to do the other's job, which keeps both short and
keeps their findings separable.

## How the two lines differ

The DSP line (e.g. `release/dsp/v1.1.3`) differs from the main line in ways that matter; the
agents' main-line notes cover the differences below.

| | main line (`~/develop/savi`) | DSP line (`release/dsp/v1.1.3`) |
| --- | --- | --- |
| `CLAUDE.md` / `architecture.md` | present (no `PACKAGES.md`) | **absent** — rules are inlined into the agents |
| Process list | `savi.yaml` + `savi-prod.yaml` (both must be updated) | **`dev.app.config.js`** — one file, `daemonSrvs` + `driverProcesses` |
| pm2 | `./cli-bazel/pm2` (regenerates ecosystem from yaml) | **`./cli-bazel/pm2.sh`** (passthrough to the vendored pm2) |
| Bring-up | `./cli-bazel/pm2 start` | **`./cli-bazel/start-savi.sh`** (does `pm2 delete all` first) |
| Test macro | `savi_node_test` in some packages; specs mixed (`node:test`, jest-style, older chai) | **`savi_mocha_test` live** → `//<pkg>:unit_test` (19 packages) |
| Spec style | match the package's existing specs | **mocha + chai + sinon**, 74 specs, zero `node:test` |
| Fast local runner | `.ai-dev/tools/jest-shim/` | **none** — see "Toolchain state" below |
| `cli-bazel/test-unit.sh` | stale (lists targets that don't exist) | target list **accurate**, but its bazel runner is blocked |
| WebSocket client split | two packages | **no websocket packages** — the `websocket` sled-daemon service is the bridge |
| `/iterate` handoff skills | `.claude/commands/` + `.ai-dev/local/` | **absent** |

What is the same, and is what the invariant reviewer checks:

- Bazel root is `lerna/packages/`; `//sled-daemon` → `lerna/packages/sled-daemon/`.
- `@savi/*` names do not map to directories — find with
  `grep -rl '"name": "@savi/<pkg>"' lerna/packages/ --include='package.json'`.
- `interfaceFactory` (NATS) vs `localIntfFactory` (mock hemera, in-process), both from
  `lerna/packages/daemon/src/plugins/interface-factory.js`.
- `collectionsDirectDB` is the default for server-side code; `collections` is the
  NATS-mediated exception for browsers and A/V hardware.
- `dependsOn` in `sled-daemon/src/config.<name>.js` is startup ordering and restart
  propagation over hemera pub/sub — never an npm or IoC dependency.
- Interfaces have two sides: caller methods and handler registration methods.
- IoC `register()` only declares; resolution is lazy and deferred to `start()`.

## DSP toolchain state (last observed 2026-09-09 — re-check)

Worth knowing before you trust a green result. There are three ways to run the 74 specs and
none of them works out of the box here:

1. **`METEOR_VERSION=<v> ./deployment/run-unit-tests.sh`** — the authority, and what CI
   runs. Docker image `savicontrols/savi-dev:${METEOR_VERSION}`, does `lerna bootstrap`
   then `lerna run lint && lerna run test`. Needs `METEOR_VERSION` set (it is not in `.env`
   or `.config`) and the image pulled. Whole repo only, no scoping.
2. **`lerna run test` / mocha directly** — only `@savi/core` and `@savi/sled-core` have a
   real `test` script (`mocha --exit` over their whole subtree). Every other package is
   `echo "Error: no test specified" && exit 0`, which passes vacuously. Needs
   `lerna bootstrap` first: mocha, chai, and sinon are absent from `lerna/packages`,
   `core/`, and `sled-core/` `node_modules` in this clone.
3. **`bazel test //<pkg>:unit_test`** — the 19 targets exist, but bazel fails at repository
   fetch: this branch pins `build_bazel_rules_nodejs` to `savicontrols/rules_nodejs
   v1.4.0-savi1`, whose GitHub release asset now returns 404.

   ```
   ERROR: no such package '@build_bazel_rules_nodejs//': ... GET returned 404 Not Found
   ```

`spec-runner` and `spec-author` are written to name which path they used and to report
"could not run" rather than improvise. That is deliberate: a hand-rolled harness with
missing chai matchers passes *vacuously*, and a false green here is worse than no answer.

If you want a fast local loop, the main-line checkout solves it with
`.ai-dev/tools/jest-shim/`, but that shim provides jest globals and these specs are chai —
it would need a chai-shaped equivalent. Ask if you want one built.
