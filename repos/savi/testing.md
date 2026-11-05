# Savi testing

Read by `spec-runner`, `spec-author` and `work-ticket`. Detect the line first (see
`README.md`). Facts with a date are *observed*, not guaranteed.

## Main line

- **Runner:** `./cli-bazel/bazel test //<pkg>:<target>`. Take the target name from the
  package's `BUILD.bazel`; don't assume `:unit_test`. `cli-bazel/test-unit.sh` lists
  targets but may be stale; verify each exists.
- **Fast loop:** `node .ai-dev/tools/jest-shim/run-spec.js <abs-path>` runs jest-style
  driver specs without bazel, and works in worktrees where bazel hangs. It lacks
  `jest.mock`: report a spec that needs it as "needs bazel", not as a failure.
- **Style:** match the nearest spec in the same package. `node:test` +
  `node:assert/strict` where the package uses `savi_node_test`; jest-style
  (`describe`/`it`/`expect`/`jest.fn`) for driver specs; mocha/chai only where the package
  still uses it. No specs in the package: follow `CLAUDE.md`, else ask.
- Confirm a new spec is picked up by reading the target's `srcs`/glob in `BUILD.bazel`.

## DSP line

All 74 specs are **mocha + chai** (`describe`/`it`, `chai.expect`, `chai-as-promised`,
`sinon`, `mock-require`). Zero `node:test`. Three runner paths, none quick (observed
2026-09-09):

1. **`METEOR_VERSION=<v> ./deployment/run-unit-tests.sh`**: what CI runs, and the
   authority. Docker image `savicontrols/savi-dev:${METEOR_VERSION}`: `bin/tools-setup`,
   rsync to `/tmp/savi/lerna`, `lerna bootstrap`, then `lerna run lint` and
   `lerna run test` with a fixed ignore list (`savi-ui`, `shef`, `cloud-daemon`,
   `sled-daemon`, `daemon`, `webos3`). Needs `METEOR_VERSION` set (not in `.env` or
   `.config`) and the image pulled. Whole repo only.
2. **`lerna run test` / mocha directly.** Only `@savi/core` (`lerna/packages/core`) and
   `@savi/sled-core` have a real `test` script (`mocha --exit "./{,!(node_modules)/**/}*.spec.js"`
   over the whole subtree). **Every other package's script is
   `echo "Error: no test specified" && exit 0`: it passes vacuously.** Needs
   `lerna bootstrap` first; mocha, chai and sinon are absent from a fresh clone. Check:
   `ls lerna/packages/core/node_modules/.bin/mocha`.
3. **`bazel test //<pkg>:unit_test`.** 19 `BUILD.bazel` files call `savi_mocha_test`
   (`//bazel/js_library:mocha.bzl`); `grep "'//" cli-bazel/test-unit.sh` lists them
   accurately. But bazel fails at repository fetch: the branch pins
   `build_bazel_rules_nodejs` to `savicontrols/rules_nodejs v1.4.0-savi1`, whose release
   asset 404s:
   `ERROR: no such package '@build_bazel_rules_nodejs//': ... GET returned 404 Not Found`.
   If it works for you, a warm cache or a mirror is doing it; say which.

Every path runs whole subtrees, so expect unrelated failures in an area that was already
red. Establish the baseline (stash, re-run, compare) before blaming the diff.

**Vacuous bucket:** a package whose `test` script is the `echo ... exit 0` stub is
*untested*, never a pass. **No target:** a package with specs whose `savi_mocha_test` is
commented out. Don't uncomment it as a side effect (it pulls in test deps and can break the
build); report it. A spec outside the `core`/`sled-core` trees is never reached by
`lerna run test`, so CI never runs it; say so.

### DSP spec house style

Model on `lerna/packages/core/utils/fifo-queue.spec.js`:

```js
'use strict'

const ThingKlass = require('./thing')
const chai = require('chai')
const chaiAsPromised = require('chai-as-promised')
chai.use(chaiAsPromised)
const expect = chai.expect

describe('thing', function() {
    it('should reject when the queue is full', async function() {
        const thing = new ThingKlass({maxCount: 1})
        await expect(thing.proc(...)).to.eventually.be.rejectedWith('job has been droped')
    })
})
```

`'use strict'`, 4-space indent, no semicolons, trailing commas in multi-line literals,
`function()` callbacks in older specs (newer ones use arrows; match the file),
`chai-as-promised` for async assertions. Available (`devPackages` in
`lerna/packages/bazel/js_library/mocha.bzl`): `chai`, `chai-as-promised`, `sinon`,
`sinon-chai`, `mock-require`. **Never write `node:test` here**: the mocha glob picks it up
and it fails. Place `foo.spec.js` next to `foo.js`.

`sinon.useFakeTimers()` only when the module is time-driven, always restored in
`afterEach`: `savi_mocha_test` runs the whole package in one process, so a leaked clock
corrupts every later spec.

## Both lines

- Don't unit-test the mesh. If a unit needs hemera, Mongo or a real device, inject the
  dependency and stub it, or make it an end-to-end check (`runtime.md`). `interfaceFactory`
  interfaces are injectable for exactly this reason.
- Async device I/O (queues, retries, rate limiters, reconnects) is where the bugs are;
  test error and timeout paths before the happy path.
- If you want a fast DSP loop, the main line's jest shim is the model, but these specs are
  chai, so it would need a chai-shaped equivalent. Ask before building one.
