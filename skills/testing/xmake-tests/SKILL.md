---
name: xmake-tests
description: Use when setting up or running tests in an Xmake project — declaring test cases with `add_tests`, running them via `xmake test`, and wiring unit-test frameworks like gtest/catch2/doctest.
---

# Xmake Tests

Xmake has built-in test support via `add_tests` on a target. `xmake test` builds and runs all declared test cases.

## Basic test on an executable

```lua
target("app_test")
    set_kind("binary")
    add_files("tests/*.cpp", "src/lib/*.cpp")
    add_tests("default")                                     -- run the binary with no args
    add_tests("with_arg", {runargs = {"--quick"}})
    add_tests("expect_zero", {trim_output = true, pass_outputs = "OK"})
    add_tests("fail_case",  {fail_outputs = {"ERROR", "FATAL"}})
```

Running:

```bash
xmake test                         # run all tests
xmake test app_test/default        # run one case
xmake test -v                      # show output
xmake test -jN                     # parallel
```

Options per case:

- `runargs` — arguments to pass
- `rundir` — working directory
- `runenvs` — environment variables (table)
- `trim_output` — trim stdout before comparison
- `pass_outputs` / `fail_outputs` — substring or list of substrings that decide pass/fail
- `group` — tag tests into groups, e.g. `group = "unit"` → `xmake test -g unit`

## Using a test framework

### doctest / catch2 (header-only)

```lua
add_requires("doctest")

target("unit")
    set_kind("binary")
    add_files("tests/*.cpp")
    add_packages("doctest")
    add_tests("default")
```

### gtest

```lua
add_requires("gtest", {configs = {main = true}})

target("unit")
    set_kind("binary")
    add_files("tests/*.cpp", "src/lib/*.cpp")
    add_packages("gtest")
    add_tests("default")
```

## Auto-discovered test targets

A common convention: one test target per file, auto-generated.

```lua
for _, file in ipairs(os.files("tests/test_*.cpp")) do
    local name = path.basename(file)
    target(name)
        set_kind("binary")
        set_default(false)                 -- don't build with `xmake`
        add_files(file, "src/lib/*.cpp")
        add_tests("default")
end
```

`set_default(false)` keeps test binaries out of the default build; `xmake test` still picks them up.

## CI usage

```bash
xmake f -m release --yes
xmake build
xmake test -v
```

`--yes` auto-accepts package prompts.

## When to branch out

- Adding the test framework as a package → `xmake-packages`
- Running tests under a cross-toolchain emulator → `xmake-toolchains`
