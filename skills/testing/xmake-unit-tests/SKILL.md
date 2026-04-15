---
name: xmake-unit-tests
description: Use when running or adding **xmake's own** unit tests — working in the `xmake-io/xmake` source tree, invoking `xmake l tests/run.lua [testname]`, writing new test files under `tests/`, and integrating with `srcenv.profile`. Not for testing *your own* project — that's `xmake-tests`.
---

# Running & Writing Xmake's Unit Tests

This skill is about the **xmake project's own** internal test suite — useful when you are fixing a bug in xmake itself, adding a new API, or verifying a backport. For testing your own C/C++ project, see `xmake-tests`.

The suite lives under `tests/` in the xmake source tree. Each subdirectory is a self-contained test (usually a mini project) driven by a `runner.lua` or a `test.lua`.

## Prerequisites

You need a checkout of xmake and the source-level env loaded:

```bash
git clone --recursive https://github.com/xmake-io/xmake.git
cd xmake
./configure && make -j            # or on Windows: cd core && xmake
source scripts/srcenv.profile     # so `xmake` points at the local checkout
xmake l xmake.programdir          # verify: should end in .../xmake/xmake
```

See `xmake-dev` for the full build-from-source flow.

## Run the whole suite

```bash
xmake l tests/run.lua
```

Iterates every subdirectory under `tests/` that contains a `runner.lua` and runs it. Expect a scroll of "pass/skip" output.

## Run a single test or group

```bash
xmake l tests/run.lua actions/build
xmake l tests/run.lua apis/add_files
xmake l tests/run.lua modules/lib_detect_find_tool
xmake l tests/run.lua plugins/project
xmake l tests/run.lua projects/c/console
```

The argument is a relative path under `tests/`. Any sub-tree works:

- `actions/` — tests for `xmake build`, `xmake run`, etc.
- `apis/` — per-API tests (`add_files`, `add_defines`, `target:add`, …)
- `modules/` — library module tests (`lib.detect.find_tool`, `net.http`, …)
- `plugins/` — plugin tests (project generator, doxygen, …)
- `projects/` — end-to-end mini projects per language
- `cli/` — CLI flag tests
- `benchmarks/` — performance benchmarks

## Verbose and diagnosis

```bash
xmake l -vD tests/run.lua actions/build
```

`-v` shows each sub-command the test runs; `-D` dumps a Lua traceback on failure. Use this whenever a test fails — the default output is terse.

## Layout of a single test

A typical test directory:

```
tests/apis/add_files/
├── runner.lua        # entry point
└── (optional support files, sample sources, etc.)
```

`runner.lua`:

```lua
-- tests/apis/add_files/runner.lua
function main(t)
    -- t is a context table with helpers (tmpdir, assertions, etc.)
    local tmpdir = os.tmpdir()
    os.cd(tmpdir)
    os.mkdir("src")
    io.writefile("src/main.cpp", [[int main(){ return 0; }]])
    io.writefile("xmake.lua", [[
        target("demo")
            set_kind("binary")
            add_files("src/*.cpp")
    ]])
    os.exec("xmake -r")
    assert(os.isfile("build/" .. os.host() .. "/" .. os.arch() .. "/release/demo"))
end
```

Conventions:

- Exported `main(t)` is the entry point. Any uncaught error (including `assert` failures) marks the test as failed.
- Use `os.exec` for commands that must succeed (it raises on non-zero).
- Use `os.iorun` / `os.iorunv` when you need to capture output.
- Use `raise("message")` for explicit failures with a custom message.
- Each test should clean up after itself or at least write into `os.tmpdir()`.

## Adding a new test

1. **Pick the right bucket**: API tests go under `tests/apis/<name>/`; action tests under `tests/actions/<action>/<case>/`; module tests under `tests/modules/<module>/`; end-to-end projects under `tests/projects/<lang>/<case>/`.
2. **Create the directory** with a `runner.lua`.
3. **Keep it hermetic**: no hard-coded paths outside `os.tmpdir()`, no network unless required, no assumptions about parallelism.
4. **Run it locally**:
   ```bash
   xmake l -vD tests/run.lua apis/mynew
   ```
5. **Run the whole suite** before submitting to make sure you didn't regress anything:
   ```bash
   xmake l tests/run.lua
   ```
6. Commit both the test and the fix/feature it covers. Same PR.

## Iteration workflow

```bash
source scripts/srcenv.profile

# Edit xmake Lua source (no rebuild needed)
vim xmake/actions/build/main.lua

# Rerun the specific test
xmake l -vD tests/run.lua actions/build/basic

# If native C core changed, rebuild first
make -j && xmake l tests/run.lua
```

Lua edits don't need a rebuild — `srcenv.profile` points `XMAKE_PROGRAM_DIR` at your checkout, so `xmake l tests/run.lua` always runs the Lua you just edited.

## Benchmarks

```bash
xmake l tests/run.lua benchmarks/<name>
```

Benchmarks live under `tests/benchmarks/` and print timing data instead of pass/fail. Use them when evaluating a performance change in xmake itself (see `xmake-build-optimization` for project-level optimization).

## CI hook

Xmake's CI runs `xmake l tests/run.lua` on every push across Windows / Linux / macOS. If you open a PR, read the CI logs — failures usually point at a specific subtest path you can reproduce locally with:

```bash
xmake l -vD tests/run.lua <failing-test-path>
```

## Pitfalls

- **Forgot to `source scripts/srcenv.profile`.** Tests run against the installed `xmake` instead of your checkout — Lua edits have no effect. Check with `xmake l xmake.programdir`.
- **Test mutates cwd.** `os.cd(os.tmpdir())` at the top; otherwise a later test may inherit the wrong directory.
- **Platform-specific assumptions.** `tests/` runs on all three OSes in CI. Gate platform-only cases with `is_host("windows")` etc., not hard-coded paths.
- **Flaky network calls.** Prefer stubbed/local fixtures. If a test genuinely needs the network, isolate it under `tests/net/` and tag it so CI can skip it when offline.
- **Rebuilding the C core after Lua edits "just to be safe".** Not needed — `make` only matters for C changes under `core/src/`.

## When to branch out

- Testing *your own* C/C++ project with `add_tests` / `xmake test` → `xmake-tests`
- Building xmake from source / debugging the Lua + C core → `xmake-dev`
- Profiling slow tests → `xmake-troubleshooting`
