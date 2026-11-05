# Savi QA environment

Read by `qa-ticket` and `release-regression`.

## Test server

```
savis-101627.local            (SSH, user `savi`)
http://savis-101627.local/    (web UI)
```

Use another server only when the user names one. Never production.

## Artifacts

- **MR build:** the latest successful pipeline on the MR's current HEAD commit. The sled
  package is the `package-sled` job's `artifacts/package/*`; other components have their own
  jobs (e.g. `build-dsp`). Download with the GitLab MCP.
- **Release candidate:** the latest successful pipeline on the RC branch, same job. Its
  pipeline commit must equal the RC HEAD and the version string must match the RC.

## Deploy

1. Upload to a unique path in `/tmp` on the server (`/tmp/<artifact-name>`).
2. SSH in, check `hostname` / `hostname -f`. Extract in its real format (inspect it; for a
   tar archive, `tar -xf <artifact>`), then run the installer packaged with the artifact
   with the privileges it needs. Capture output and exit status.
3. After a successful install, and after re-checking the hostname: `reboot -f` (elevated if
   needed). The SSH drop is expected.
4. Reconnect; `hostname` and `uptime` to prove it rebooted; check the services came up
   (process list from `savi.yaml` on the main line).
5. Confirm the installed version (UI version, package metadata or service info) matches the
   artifact.

## Existing fakes (reuse before building a harness)

- `lerna/packages/webos3-simulator`
- `driver-*/test-tools/` (e.g. `fake-everpass`)
- `core/interface-factory-mock`, `core/utils/mock`
- `webos3-app/src/libs/fake_scap`
- Scripted mesh clients using `interfaceFactory`, pointed at the server's NATS

Typical harnesses: simulated devices speaking a driver's protocol; extra IP addresses so
several simulated devices coexist; HTTP/TCP/UDP servers capturing what the sled sends out;
fake cloud endpoints; fault injection. Keep them in `~/qa-harness/<run>/`, never in a Savi
checkout.

## General regression areas

Derive the list from the running UI, `CLAUDE.md`/`architecture.md` and `savi.yaml`, with
extra weight on packages changed since the baseline. At minimum, when present:

- every expected process running and healthy after the upgrade and reboot
- login and authentication
- navigation to every top-level area
- device and driver discovery, control, status feedback, and reconnect
- configuration save, reload and persistence
- scheduling and events
- media, streams and displays
- the API and the browser WebSocket bridge
- upgrade from the baseline: data and settings survive
- logs free of new errors afterwards
