# Reproduction: vite-plus + @cloudflare/vitest-pool-workers crash

Minimal reproduction for [voidzero-dev/vite-plus#1076](https://github.com/voidzero-dev/vite-plus/issues/1076).

## Status: Fixed in vite-plus v0.2.0

Upgrading vite-plus `0.1.13` -> `0.2.0` fixes the crash. `vp test` now passes:

```
 Test Files  1 passed (1)
      Tests  3 passed (3)
```

v0.2.0 ([#1588](https://github.com/voidzero-dev/vite-plus/pull/1588)) **removes the rebundled `@voidzero-dev/vite-plus-test` wrapper** and runs upstream `vitest` directly, so there is no longer a second `@vitest/runner` copy to desync, the dual-instance cause described below can't happen.

Note: this project keeps `vitest` as a direct devDependency, so under pnpm it is hoisted to the top-level `node_modules` and the cloudflare pool resolves `vitest/worker` cleanly (no separate pnpm `No such module "vitest/worker"` issue).

## Bug description (vite-plus 0.1.13, historical)

When running `vp test` with `@cloudflare/vitest-pool-workers`, any test file that uses `describe()` crashes immediately with:

```
TypeError: Cannot read properties of undefined (reading 'config')
  at initSuite .../vite-plus-test/dist/@vitest/runner/index.js:1203:25
    validateTags(runner.config, suiteTags);
```

Running the same tests with `vitest run` (standard vitest) passes without error.

## Steps to reproduce

```bash
git clone <this-repo>
cd vite-plus-issue-1076
pnpm install
```

### Passes (standard vitest)

```bash
pnpm run test
# vitest run
#  PASS  test/index.test.ts (3 tests)
```

### Fails (vite-plus test)

```bash
vp test
#  FAIL  test/index.test.ts
# TypeError: Cannot read properties of undefined (reading 'config')
#   at initSuite .../vite-plus-test/dist/@vitest/runner/index.js:1203:25
#     validateTags(runner.config, suiteTags);
```

## Root cause

`VitestCoreResolver` in vite-plus intercepts the bare `"vitest"` specifier and redirects it to `vite-plus-test/dist/index.js`. This causes the Cloudflare Worker (Durable Object) to instantiate a separate `@vitest/runner` module from vite-plus-test's bundled copy.

`clearCollectorContext()` is called on the Node.js-side runner instance (setting `runner = testRunner`), but the Worker-side instance never receives this call — it runs in a separate process where `globalThis` is not shared. When the test file's `describe()` calls `initSuite()`, `runner` is still `undefined`.

```mermaid
flowchart TD
    subgraph nodejs["Node.js process (pool)"]
        VCR["VitestCoreResolver"]
        STD["@vitest/runner (standard)\ninstance A"]
        CCC["clearCollectorContext\nrunner = testRunner ✅"]

        VCR -->|"vitest/ @vitest/ → this.resolve"| STD
        STD --> CCC
    end

    subgraph worker["Cloudflare Worker process (Durable Object)"]
        VCR2["VitestCoreResolver"]
        BUNDLED["@vitest/runner (vite-plus-test bundled)\ninstance B — never initialized"]
        TEST["test/index.test.ts\ndescribe()"]
        CRASH["initSuite\nrunner = undefined → CRASH 💥"]

        VCR2 -->|"vitest → resolve(distDir, 'index.js')"| BUNDLED
        TEST -->|import vitest| VCR2
        BUNDLED --> CRASH
    end

    CCC -.->|"separate process\nglobalThis not shared"| CRASH
```

## Environment

| Package | Version |
|---|---|
| vite-plus | 0.2.0 |
| vitest | 4.1.9 |
| @cloudflare/vitest-pool-workers | 0.13.3 |
| wrangler | 4.76.0 |
| Node.js | >=18 |
