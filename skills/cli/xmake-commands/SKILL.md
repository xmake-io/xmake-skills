---
name: xmake-commands
description: Use when invoking Xmake from the command line — configuring, building, running, cleaning, installing, packing, or inspecting a project. Covers the common flags for `xmake f`, `xmake`, `xmake run`, `xmake install`, `xmake pack`, and friends.
---

# Xmake Commands

Xmake's CLI is a single `xmake` binary with subcommands. Without a subcommand it runs `build`.

## Configure (`xmake f` / `xmake config`)

```bash
xmake f -m release                # build mode
xmake f -p linux -a x86_64        # platform + arch
xmake f --toolchain=clang
xmake f -c                        # clear old config
xmake f --menu                    # interactive TUI (shows all options)
xmake f -v                        # verbose
```

Config is persisted under `.xmake/`, so later `xmake` calls reuse it.

## Build

```bash
xmake                 # build all default targets
xmake app             # build a specific target
xmake -r              # rebuild (force)
xmake -j 8            # parallel jobs
xmake -v              # show compile commands
xmake -vD             # verbose + diagnostics (for debugging xmake.lua)
xmake -w              # treat warnings
```

## Run

```bash
xmake run                  # run default runnable target
xmake run app arg1 arg2    # pass arguments
xmake run -d app           # run under debugger (gdb/lldb/cdb)
```

## Clean

```bash
xmake clean                # current config
xmake clean --all          # all configs
xmake clean app            # one target
```

## Install / uninstall

```bash
xmake install -o /usr/local
xmake install --root app       # one target, root-prefixed
xmake uninstall -o /usr/local
```

## Pack (`xmake pack`)

Produce distributable archives (zip/tar.gz/deb/rpm/nsis/wix/...):

```bash
xmake pack                           # default format
xmake pack -f zip
xmake pack -f nsis                   # Windows installer
xmake pack -f deb mylib              # Debian package for one target
```

Packaging metadata is defined via `xpack(...)` in `xmake.lua`.

## Project info & inspection

```bash
xmake show                           # project summary
xmake show -l targets
xmake show -l toolchains
xmake show -l plats
xmake show -t app                    # info for one target
xmake show --key=options
```

## Project file generation (for IDEs/other tools)

```bash
xmake project -k compile_commands    # compile_commands.json
xmake project -k cmakelists          # CMakeLists.txt (one-way export)
xmake project -k vsxmake -m "debug,release"   # Visual Studio solution
xmake project -k xcode               # Xcode project
```

## Self-update

```bash
xmake update                 # stable
xmake update -s dev          # dev branch
```

## When to branch out

- Writing tests runnable via `xmake test` → `xmake-tests`
- Packaging details and `xpack` interfaces → see xmake docs
