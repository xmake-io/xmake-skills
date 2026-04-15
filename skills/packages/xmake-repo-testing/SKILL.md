---
name: xmake-repo-testing
description: Use when adding or modifying a package recipe in the xmake-repo repository (github.com/xmake-io/xmake-repo) and you need to test-build it locally. Wraps the `xmake l scripts/test.lua` workflow with `--shallow`, platform/arch/mode/kind/configs flags so package changes can be verified quickly before submitting a PR.
---

# Testing Packages in xmake-repo

[`xmake-repo`](https://github.com/xmake-io/xmake-repo) is the official package recipe repository. Each recipe is a `packages/<a>/<name>/xmake.lua` describing how to fetch, build, and install a library. The repo ships a test script at `scripts/test.lua` that drives a full install + test build of a recipe exactly like CI does.

Always run it from the **xmake-repo checkout root**.

## The core command

```bash
xmake l scripts/test.lua -vD --shallow <package>
```

- `xmake l` — run a Lua script in the xmake environment.
- `-v` — verbose (show each sub-command).
- `-D` — diagnosis (full tracebacks on failure; essential when debugging a broken recipe).
- `--shallow` — **only install the package you asked for**, not its dependencies. Dependencies are pulled from the remote binary cache instead of rebuilt locally. This is the single most important flag for iterating quickly.
- `<package>` — the package name, e.g. `fmt`, `zlib`, `boost`.

Example — test the `fmt` recipe:

```bash
cd xmake-repo
xmake l scripts/test.lua -vD --shallow fmt
```

## Common variations

### Version / branch

Append the version to the package name, same syntax as `add_requires`:

```bash
xmake l scripts/test.lua -vD --shallow "fmt 10.2.1"
xmake l scripts/test.lua -vD --shallow "cmake master"
```

Quote it — the space matters.

### Static vs shared

```bash
xmake l scripts/test.lua -vD --shallow -k shared fmt
xmake l scripts/test.lua -vD --shallow -k static fmt
```

### Debug / release

```bash
xmake l scripts/test.lua -vD --shallow -m debug  fmt
xmake l scripts/test.lua -vD --shallow -m release fmt
```

### Platform / arch

```bash
xmake l scripts/test.lua -vD --shallow -p linux   -a x86_64 fmt
xmake l scripts/test.lua -vD --shallow -p macosx  -a arm64  fmt
xmake l scripts/test.lua -vD --shallow -p mingw   -a x86_64 --mingw=/opt/llvm-mingw fmt
xmake l scripts/test.lua -vD --shallow -p android -a arm64-v8a --ndk=~/android-ndk-r26 fmt
xmake l scripts/test.lua -vD --shallow -p iphoneos -a arm64 fmt
xmake l scripts/test.lua -vD --shallow -p wasm --toolchain=emcc fmt
```

### Package configs (recipe options)

Recipes declare options via `add_configs(...)`. Pass them with `-f`:

```bash
xmake l scripts/test.lua -vD --shallow -f "shared=true,pic=true" openssl
xmake l scripts/test.lua -vD --shallow -f "header_only=false" fmt
xmake l scripts/test.lua -vD --shallow -f "regex=true,system=true" boost
```

### MSVC runtime / toolset

```bash
xmake l scripts/test.lua -vD --shallow --runtimes=MD  fmt
xmake l scripts/test.lua -vD --shallow --runtimes=MT  fmt
xmake l scripts/test.lua -vD --shallow --vs=2022 --vs_toolset=14.38 fmt
```

### Fetch-only (no build)

Verify URL, hash, and unpack work without actually compiling:

```bash
xmake l scripts/test.lua -vD --shallow --fetch fmt
```

### Try the precompiled artifact

```bash
xmake l scripts/test.lua -vD --shallow --precompiled fmt
```

### Debugging the source build

Point at a local source tree to iterate on patches without re-downloading:

```bash
xmake l scripts/test.lua -vD --shallow -d /path/to/fmt-source fmt
```

## What the script actually does

1. Reads `packages/<a>/<name>/xmake.lua`.
2. Runs `xmake f -c` with your flags.
3. Installs the package (with `--shallow`, deps come from the remote cache).
4. Builds and runs the recipe's `on_test(...)` block against the installed package.

A successful run ends with `passed!`. A failure prints the command that failed — rerun it with `-vD` if you didn't already.

## Iteration workflow

1. `xmake l scripts/list.lua <pkg>` — see recipe metadata.
2. Edit `packages/<a>/<pkg>/xmake.lua`.
3. `xmake l scripts/test.lua -vD --shallow <pkg>` — test.
4. Adjust `add_configs`, `on_install`, `on_test` until green.
5. Repeat on other platforms if the recipe is cross-platform.

For a brand-new package, scaffold it first:

```bash
xmake l scripts/new.lua <pkg>
```

## Useful flags reference

| Flag | Purpose |
| --- | --- |
| `-v` / `-D` | verbose / diagnosis |
| `--shallow` | only install the target package, not its deps |
| `-k static\|shared` | library kind |
| `-m debug\|release` | build mode |
| `-p <plat>` / `-a <arch>` | target platform / architecture |
| `-f "k=v,k=v"` | package configs (recipe options) |
| `--runtimes=MD\|MT\|MDd\|MTd` | MSVC runtime |
| `--vs=`, `--vs_toolset=`, `--vs_sdkver=` | VS version pinning |
| `--ndk=`, `--ndk_sdkver=` | Android NDK |
| `--sdk=`, `--mingw=` | cross-toolchain roots |
| `--toolchain=<name>` | force a toolchain |
| `--fetch` | fetch/unpack only, no build |
| `--precompiled` | install from binary cache |
| `--remote` | run test on the remote server |
| `-d <dir>` | use a local source directory (debug patches) |

## Common pitfalls

- **Forgetting `--shallow`.** Without it, the script rebuilds every transitive dependency from source — turning a 10-second iteration into 30 minutes.
- **Running from the wrong directory.** The script must be invoked from the `xmake-repo` checkout root so `scripts/test.lua` and `packages/` resolve correctly.
- **Stale cache hiding your change.** Force a clean build with `xmake require --force <pkg>` or delete `~/.xmake/packages/<a>/<pkg>/<version>/<hash>` between runs if a recipe change does not seem to take effect.
- **Quoting version strings.** `fmt 10.2.1` must be one quoted argument, otherwise the shell splits it and `10.2.1` is parsed as a separate flag.

## When to branch out

- Consuming packages from a *project* → `xmake-packages`
- Cross-compilation setup → `xmake-toolchains`
