# Savi architecture rules

Read by `conventions-reviewer`, `impact-tracer`, `async-bug-hunter` and `work-ticket`. On
the main line, the repo's `CLAUDE.md` and `architecture.md` win wherever they state a rule;
use their wording. On the DSP line this file is the only written copy.

Savi's service mesh hides its own topology on purpose: a caller can't tell whether a
service runs in-process, in another Node process, on another machine, or in a browser tab.
That's good for the code and bad for grep.

## Finding paths

Bazel root is `lerna/packages/`; `//sled-daemon` → `lerna/packages/sled-daemon/`.
`@savi/*` names do **not** map mechanically to directories:

```sh
grep -rl '"name": "@savi/<pkg>"' lerna/packages/ --include='package.json'
```

then append the suffix from the `require()`: `require('@savi/daemon/src/index')` →
`lerna/packages/daemon/package.json` → `lerna/packages/daemon/src/index.js`. Never name a
path you haven't opened.

## House rules (numbered; reviewers cite them)

**1. Data access: `collectionsDirectDB` is the default.** MongoDB on the sled is bound to
localhost, so every server-side process can connect directly. Use
`keys.dataDefinitionsI.collectionsDirectDB` for all server-side code.
`keys.dataDefinitionsI.collections` is DB access mediated by the data service over NATS:
only where the consumer can't hold a direct connection (browsers, embedded A/V hardware).
Flag a new server-side plugin resolving `collections`.
Don't flag a service registering both `_keys.services` and `_keys.servicesRemoteDB` via
the `helper(collectionKey)` pattern. That's deliberate: plugins in the same process may
resolve either path, and the wiring belongs to the plugin author.

**2. `interfaceFactory` vs `localIntfFactory`.** Both come from
`lerna/packages/daemon/src/plugins/interface-factory.js`.
`keys.interfaceFactory.interfaceFactory` is real NATS: crosses processes and machines,
reachable from browsers via the `websocket` service. `keys.interfaceFactory.localIntfFactory`
is mock hemera: in-process, no network, no serialisation. `localIntfFactory` is correct
only when the service always runs in-process and holds no in-memory state another process
must observe (all state in Mongo is fine). Flag a local service with shared in-process
state, and one whose interface and implementation are wired in *different* plugins. The
canonical pattern wires both inside the same `ioc.singleton`.

**3. `dependsOn` is not a dependency declaration.** In
`lerna/packages/sled-daemon/src/config.<name>.js`, `dependsOn` is startup ordering and
restart propagation over hemera pub/sub (implemented in `lerna/packages/daemon/src/index.js`):
- before `start()`, the process blocks until every named service broadcasts
  `savi.daemonReady.<name>`;
- after `start()`, it broadcasts its own readiness on a 1-second interval;
- if a dependency broadcasts a newer start timestamp (it restarted), the process exits so
  pm2 restarts it in order.

Flag a `dependsOn` entry added to express an npm import or IoC need, and any comment or
commit message that infers one from `dependsOn`. A rename breaks it silently: the
dependent blocks forever.

**4. Interfaces have two sides.** An interface from `interfaceFactory` exposes **caller
methods** for consumers and **handler registration methods** for the implementation. Both
inject the same interface, so a `require` tells you nothing about which side a file is on.
Flag any claim that an interface plugin is only needed where the service runs.

**5. IoC registration is lazy.** All plugin `register()` functions run in parallel at
startup and only *declare* singletons. Resolution happens in `ioc.use(key)` inside a
singleton factory during `start()`, so referencing another plugin's keys in `register()` is
safe. Flag ordering hacks, retry loops or defensive awaits that work around an order that
can't happen.
The **base** plugin `lerna/packages/daemon/src/plugins/data-definitions-interface.js`
exports keys with an intentionally empty `register: () => {}`. The real singletons are
registered by `lerna/packages/sled-daemon/src/plugins/data-definitions-interface.js`, which
is supposed to have a body. Flag an implementation added to the base one.

**6. Transport is an abstraction.** Hemera has real NATS (`keys.hemera.hemera`), mock
(`keys.hemeraMock.hemera`) and a WebSocket path for browsers and A/V hardware behind one
interface. Flag service code that branches on transport, sniffs for NATS, or assumes a call
is cheap because it is "local"; only `localIntfFactory` is in-process.

**7. A new process must be registered.**
- DSP line: `lerna/packages/sled-daemon/src/config.<name>.js` **and** a
  `daemonSrvs.push({name: '<name>'})` line in `dev.app.config.js` (or a `driverProcesses`
  entry for a driver). Check `pm2.savi-dev.config.js` if mongo or nats is touched.
- Main line: the config **and** entries in both `savi.yaml` and `savi-prod.yaml`.

One without the other is a definite violation, not a nit.

## Blast radius: the seven planes

For a changed interface, service, IoC key, config or package, sweep all seven and say when
one is empty:

1. **Interface definition**: the interface package (typically `.../<name>/interface/index.js`);
   caller methods vs handler registration methods.
2. **Handler side**: the service(s) calling the registration methods, including mocks and
   test doubles.
3. **Caller side**: greps for the method names and requires of the interface package; read
   each call to tell the side.
4. **IoC keys**: the key namespace, and every `ioc.use(...)`/`ioc.singleton(...)` naming it.
   A `register()`-time reference is not a runtime dependency.
5. **Transport**: which factory backs it. If NATS, check browser/A/V callers under `app/`,
   `app-meteors/` and `webos3-app/`, and say if the surface is effectively public.
6. **Process configs**: `lerna/packages/sled-daemon/src/config.*.js` for the plugin and
   service name, and `dependsOn` mentions.
7. **Process list**: DSP `dev.app.config.js` (`daemonSrvs`, `driverProcesses`),
   `pm2.savi-dev.config.js` for mongo/nats, and `cli-bazel/start-savi.sh`'s explicit build
   list. Main line `savi.yaml` **and** `savi-prod.yaml`.

Also: specs (`grep -rl '<name>' --include='*.spec.js' lerna/packages/`, and whether the
package's test macro is live, see `testing.md`); consumers resolving `collections` (browser
and hardware paths widen the radius); `lerna/packages/driver-*` is a large catalogue, so a
change to a shared driver utility or the command router can touch dozens. Count them.

Silent-breakage risks (no build or lint signal): `dependsOn` names, process-list entries,
string-keyed hemera topics, browser callers.

## Async hotspots

The bugs that reach production here are lifetime, ordering and partial-failure bugs in
async code that looks fine. On top of the generic classes in `async-bug-hunter`:

- An unhandled rejection takes the process down, and the pm2 restart cascades to everything
  listing it in `dependsOn`.
- The daemon broadcasts readiness on a 1-second interval; a duplicated interval is not
  harmless.
- Listeners registered on every connect but removed on no disconnect are very common in the
  driver catalogue.
- A hemera handler that throws instead of returning an error: check what the caller sees
  across the transport boundary.
- Blocking work in a handler that also serves browser clients through `websocket`.

## Tracer-bullet example

A typical golden path for a UI-driven device change (names illustrative):

```
User picks a region state in the layout editor
  → LayoutRegionPanel → canvas interface caller method
  → websocket bridge → NATS        [RISK: driver restarting; dependsOn gating]
  → canvas display driver handler (registered via interfaceFactory)
  → driver → device command         [RISK: device disconnects mid-command]
  → device confirms → driver publishes new state
  → UI subscription → panel shows the confirmed state
```

Boundaries to check on top of the generic ones: transport choice, `dependsOn` restart
ordering, and which process the handler is registered in.
