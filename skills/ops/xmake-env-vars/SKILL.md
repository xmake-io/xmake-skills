---
name: xmake-env-vars
description: Use when configuring xmake via environment variables — `XMAKE_PROGRAM_DIR`/`XMAKE_PROGRAM_FILE` (local source debug), `XMAKE_GLOBALDIR`/`XMAKE_CONFIGDIR` (config locations), `XMAKE_COLORTERM`/`XMAKE_PROFILE`, `XMAKE_RCFILES` (global defaults), `XMAKE_ROOT`, and how to list/inspect them with `xmake show -l envs`.
---

# Xmake Environment Variables

Xmake reads a handful of `XMAKE_*` environment variables to relocate config, enable profiling, load user-wide defaults, and more. They take precedence over the matching `xmake g` global setting when both are present.

## List them all

```bash
xmake show -l envs
```

Shows every recognized variable, its description, and the currently resolved value on your system.

## Reference

### Paths

| Variable | Purpose | Default |
| --- | --- | --- |
| `XMAKE_PROGRAM_DIR` | Where xmake's Lua scripts live | xmake install dir |
| `XMAKE_PROGRAM_FILE` | Which `xmake` binary to use | resolved from PATH |
| `XMAKE_GLOBALDIR` | Root for `xmake g` global config (creates `.xmake/` inside) | `~` |
| `XMAKE_CONFIGDIR` | Where per-project `.xmake/` cache goes | project dir |
| `XMAKE_TMPDIR` | Temp scratch | `/tmp/.xmake` / `%TEMP%\.xmake` |
| `XMAKE_RAMDIR` | Optional ramdisk for temp files — fastest | unset |
| `XMAKE_PKG_INSTALLDIR` | Package install root | `$XMAKE_GLOBALDIR/.xmake/packages` |
| `XMAKE_PKG_CACHEDIR` | Package cache dir | `$XMAKE_GLOBALDIR/.xmake/cache` |
| `XMAKE_LOGFILE` | Log everything xmake does to this file | unset |

Typical uses:

```bash
# Use a local source checkout of xmake (developing xmake itself — see xmake-dev skill)
export XMAKE_PROGRAM_DIR=/path/to/xmake/xmake
export XMAKE_PROGRAM_FILE=/path/to/xmake/core/build/xmake

# Keep project .xmake out of the repo root
export XMAKE_CONFIGDIR=/tmp/myproject-xmake

# Put packages on a fast disk
export XMAKE_PKG_INSTALLDIR=/mnt/fast/xmake-packages
export XMAKE_PKG_CACHEDIR=/mnt/fast/xmake-cache

# Ram-disk temp for crazy-fast configures
export XMAKE_RAMDIR=/dev/shm/xmake
```

### Output / behavior

| Variable | Values | Purpose |
| --- | --- | --- |
| `XMAKE_COLORTERM` | `nocolor` / `color8` / `color256` / `truecolor` | Force color mode |
| `XMAKE_PROFILE` | `stuck` / `trace` / `perf:call` / `perf:tag` / `perf:process` | Profiling / stuck debug — see `xmake-troubleshooting` |
| `XMAKE_ROOT` | `y` | Allow running as root (off by default — unsafe) |
| `XMAKE_LOGFILE` | path | Mirror xmake output to a log file |
| `XMAKE_THEME` | theme name | Equivalent to `xmake g --theme=...` |

### User-wide defaults: `XMAKE_RCFILES`

Xmake loads a user-specified "rc" file before any `xmake.lua` is parsed. Use it for global toolchains, default repos, company-wide policies — anything you want every project on the machine to pick up.

```bash
# Unix: colon-separated multiple files
export XMAKE_RCFILES=~/.xmake/xmakerc.lua:/etc/xmake/xmakerc.lua

# Windows: semicolon-separated
set XMAKE_RCFILES=C:\xmake\xmakerc.lua;D:\proj\xmakerc.lua
```

Example `~/.xmake/xmakerc.lua`:

```lua
-- Always enable debug+release modes
add_rules("mode.debug", "mode.release")

-- Company-wide package repo
add_repositories("mycompany https://github.com/mycompany/xmake-repo.git")

-- Pin a preferred toolchain
toolchain("mygcc")
    set_kind("standalone")
    set_toolset("cc", "gcc-11")
    set_toolset("cxx", "g++-11")
toolchain_end()
```

Precedence: rc files load before the project `xmake.lua`, so the project can still override anything they set.

## Precedence rules

For settings that have both an env var and a `xmake g` equivalent:

1. Env var (highest)
2. `xmake g` global config
3. Project-level `xmake f` config
4. Built-in default

So `XMAKE_COLORTERM=nocolor xmake` always disables color, regardless of `xmake g --theme`.

## Inspecting from scripts

From inside a script-domain hook:

```lua
on_load(function (target)
    local configdir = os.getenv("XMAKE_CONFIGDIR")
    if configdir then
        print("using config dir %s", configdir)
    end
end)
```

`os.getenv` is one of the few env-reading APIs allowed in the description domain as read-only — fine for conditional config flags.

## Common uses

```bash
# CI: disable color, log everything
XMAKE_COLORTERM=nocolor XMAKE_LOGFILE=build.log xmake

# Debug a stuck build
XMAKE_PROFILE=stuck xmake f -c

# Profile configure
XMAKE_PROFILE=perf:process xmake f -c

# Dev xmake from a checkout
source scripts/srcenv.profile         # sets XMAKE_PROGRAM_DIR + _FILE

# Cache-aware CI
XMAKE_PKG_CACHEDIR=$HOME/ci-cache/xmake-pkg xmake
```

## Pitfalls

- **Forgetting `XMAKE_PROGRAM_DIR` after a dev test.** Leaves `xmake` pointing at a stale checkout. `unset XMAKE_PROGRAM_DIR` to restore.
- **`XMAKE_RCFILES` breaks projects.** A rc file that force-adds rules/toolchains can clash with a project that configures its own. Scope rc files carefully; prefer description-only.
- **`XMAKE_ROOT=y`.** Works, but running as root is dangerous — prefer a regular user.
- **Env var precedence over `xmake g`.** If `xmake g --color256` seems ignored, check whether `XMAKE_COLORTERM` is exported.
- **Logfile silently filling the disk.** `XMAKE_LOGFILE` appends; rotate it yourself if enabled permanently.

## When to branch out

- Developing xmake source and switching `XMAKE_PROGRAM_*` → `xmake-dev`
- Profiling / stuck debug (`XMAKE_PROFILE`) → `xmake-troubleshooting`
- Color / theme knobs → `xmake-theme`, `xmake-color-output`
