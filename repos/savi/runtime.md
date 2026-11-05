# Savi runtime: the dev sled

Read by `work-ticket` and `review-follow-up` when a change needs proving in the running
system rather than in specs: hemera topics, IoC resolution order, the WebSocket bridge, pm2
startup ordering. Those pass every unit test and fail on the sled.

## Ground rules

- **Never start, restart, stop or delete a process without explicit permission.** The dev
  sled is a shared, stateful process set with a real Mongo behind it. A restarted
  dependency broadcasts a newer start timestamp, and every process listing it in
  `dependsOn` exits so pm2 can restart it in order.
- **Attach before you launch.** Check what is already running first.
- End every report with a `LEFT RUNNING:` line: state you changed and did not restore.

## Attaching

pm2 goes through the repo wrapper, which resolves the vendored node and pm2 (via
`cli-bazel/_lib_check.sh`). A bare `pm2` is the wrong one or absent.

| | DSP line | main line |
| --- | --- | --- |
| wrapper | `./cli-bazel/pm2.sh list` / `logs <name> --lines 100 --nostream` / `describe <name>` | `./cli-bazel/pm2 …` (regenerates the ecosystem from `savi.yaml`) |
| process set | `dev.app.config.js`: `services` (mongodb, nats, app, grafana, loki, promtail), `daemonSrvs` (`data-stream`, `data`, `auth`, `api`, `canvas`, `websocket`, `metal`, `driver-command-router`, `driver-registry`, `proxy`, …), `driverProcesses`; `pm2.savi-dev.config.js` for mongo/nats | `savi.yaml` |
| bring-up (ask first) | `./cli-bazel/start-savi.sh`: builds nats, webos3-app, sled-daemon and the Meteor app, and does `pm2 delete all` first. Slow. | see `CLAUDE.md` |

Ports come from `SAVI_PORT_*` env vars (`.env` and the port block in `dev.app.config.js`);
read them. Grafana is 13000 and Loki 3100 by default.

## Browser flows

Use Playwright against the Meteor app. Prefer `browser_snapshot` (accessibility tree,
stable refs) over screenshots for finding elements; screenshot when visual evidence is
needed. Always collect console messages (client errors, WebSocket transport errors) and
network requests (failed calls, the WebSocket upgrade).

The browser is a mesh participant, not just a viewer: it calls server-side services and
registers its own services that server code calls, all through the `websocket` service.
A browser-side failure can be a server-side symptom and vice versa; check both sides.

## Mesh flows without a browser

Read the interface package for the method names, then drive it from a small script in the
scratchpad using the same `interfaceFactory` the services use, pointed at the dev NATS.
Don't invent an HTTP endpoint: most of this system is not HTTP. `api`, `apollo-server` and
`proxy` are the services that speak HTTP; check their configs for ports.

## Correlating failures

Tail the logs of every service on the path before concluding anything
(`… logs <service> --lines 200 --nostream | tail -60`). Shapes worth naming:

- Stuck before `start()`: blocked waiting for `savi.daemonReady.<name>` from something in
  its `dependsOn` (`lerna/packages/sled-daemon/src/config.<name>.js`). Ordering, not a bug.
- Exits repeatedly shortly after starting: probably seeing a newer start timestamp from a
  restarted dependency. Also ordering.
- A call that hangs with no error: usually a topic mismatch; nobody registered the handler.
