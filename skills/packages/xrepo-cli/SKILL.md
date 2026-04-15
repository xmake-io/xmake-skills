---
name: xrepo-cli
description: Use when installing, inspecting, or exporting C/C++ packages from the command line with `xrepo` — xmake's standalone package manager. Covers install/search/info/fetch/export/download, repository management, and package virtual environments (`xrepo env`).
---

# Xrepo CLI

`xrepo` is xmake's standalone C/C++ package manager. It shares the xmake runtime but is a full package tool on its own — use it outside of any project to install, query, or enter an environment with C/C++ libraries and tools.

> `xrepo` manages **global** repositories and installed packages. `xmake repo` is a *project-local* variant. If you want the change to apply everywhere on your machine, use `xrepo`.

Packages come from the official [xmake-repo](https://github.com/xmake-io/xmake-repo) by default, plus any private repos you register, plus bridges to vcpkg/brew/conan/pacman/apt etc.

## Installing packages

```bash
xrepo install zlib tbox
xrepo install "zlib 1.2.x"          # semver range — QUOTE the spec
xrepo install "zlib >=1.2.0"
```

### Mode / kind

```bash
xrepo install -m debug zlib          # debug build
xrepo install -m release zlib
xrepo install -k shared zlib         # dynamic library
xrepo install -k static zlib
```

### Platform / arch

```bash
xrepo install -p iphoneos -a arm64 zlib
xrepo install -p android -a arm64-v8a --ndk=~/android-ndk-r26 zlib
xrepo install -p mingw  --mingw=/opt/llvm-mingw zlib
xrepo install -p cross  --sdk=/opt/arm-linux-musleabi-cross zlib
xrepo install -p wasm   --toolchain=emcc zlib
```

### Package configs

Pass recipe options via `-f`:

```bash
xrepo install -f "runtimes='MD'" zlib
xrepo install -f "regex=true,thread=true" boost
xrepo install -f "shared=true,pic=true" openssl
```

### From third-party package managers

xrepo can install through other package managers and still hand you xmake-style link info:

```bash
xrepo install brew::zlib
xrepo install vcpkg::zlib
xrepo install conan::zlib/1.2.11
xrepo install pacman::zlib
xrepo install apt::zlib1g-dev
```

### Force / clean

```bash
xrepo install --force zlib           # rebuild even if cached
xrepo remove zlib                    # uninstall
xrepo clean                          # clean caches
xrepo clean --all
```

## Searching and inspecting

```bash
xrepo search zlib "pcr*"
xrepo info zlib
```

`xrepo info` shows description, versions, upstream URLs, hashes, supported platforms, and available configs — use it to discover which `-f "..."` options a package accepts before installing.

## Getting link / cflags info (`xrepo fetch`)

`fetch` resolves an installed package and prints the info you would otherwise get through `add_packages`. Great for dropping xrepo packages into a non-xmake build.

```bash
xrepo fetch pcre2                    # Lua table (linkdirs/links/defines/includedirs)
xrepo fetch --cflags openssl
xrepo fetch --ldflags openssl
xrepo fetch --cflags --ldflags conan::zlib/1.2.11
xrepo fetch -p iphoneos --cflags "zlib 1.2.x"
```

Pipe into a Makefile / CMake toolchain:

```bash
CFLAGS="$(xrepo fetch --cflags openssl)"
LDFLAGS="$(xrepo fetch --ldflags openssl)"
```

## Downloading source only

```bash
xrepo download zlib
xrepo download "zlib 2.x"
xrepo download -o /tmp zlib          # output directory
```

Does *not* build or install — just fetches and extracts the upstream source.

## Exporting an installed package

Copy an installed package (headers, libs, cmake files, …) into a standalone directory so you can ship it or hand it to another toolchain:

```bash
xrepo export -o /tmp/output zlib
xrepo export -o /tmp/output -p android -a arm64-v8a "zlib 1.2.x"
```

## Repository management

```bash
xrepo add-repo myrepo https://github.com/mygroup/myrepo
xrepo add-repo myrepo https://github.com/mygroup/myrepo dev   # branch
xrepo rm-repo myrepo
xrepo list-repo
xrepo update-repo                    # pull the latest recipes
```

Private, air-gapped setups typically: `add-repo` an internal mirror, then install everything from there.

## Package virtual environments

`xrepo env` drops you into a shell (or runs a single command) with selected packages on `PATH` / `LD_LIBRARY_PATH` / `PKG_CONFIG_PATH`. This is a sizable topic on its own — see the **`xrepo-env`** skill for ad-hoc, project-local (`./xmake.lua`), and named environments, plus CI usage and `--show` debugging.

## Quick reference

| Command | Purpose |
| --- | --- |
| `xrepo install <pkg>` | install a package |
| `xrepo remove <pkg>` | uninstall |
| `xrepo update <pkg>` | update to newer version |
| `xrepo search <pattern>` | search available recipes |
| `xrepo info <pkg>` | show recipe metadata + configs |
| `xrepo fetch [--cflags/--ldflags] <pkg>` | print link/compile info |
| `xrepo export -o <dir> <pkg>` | export installed package |
| `xrepo download [-o dir] <pkg>` | fetch source only |
| `xrepo add-repo / rm-repo / list-repo / update-repo` | global repo management |
| `xrepo env ...` | virtual environments — see `xrepo-env` skill |
| `xrepo clean [--all]` | clean caches |

## Common flags

- `-m debug|release` — build mode
- `-k static|shared` — library kind
- `-p <plat>` / `-a <arch>` — target platform / arch
- `-f "k=v,k=v"` — recipe configs (per package)
- `--runtimes=MD|MT|MDd|MTd` — MSVC runtime
- `--ndk=`, `--sdk=`, `--mingw=`, `--vs=` — toolchain roots
- `--toolchain=<name>` — force a specific toolchain
- `-v` / `-vD` — verbose / diagnosis

## Pitfalls

- **Unquoted version strings.** `xrepo install zlib 1.2.x` parses `1.2.x` as a second package name. Always quote: `"zlib 1.2.x"`.
- **`xrepo` vs `xmake repo`.** `xrepo` is global; `xmake repo` is for project-local repos inside an `xmake.lua`. They are not interchangeable.
- **Forgetting to update repos.** If a recipe was added upstream after your last sync, `xrepo update-repo` before `install`.
- **Exported/fetched paths are absolute.** `xrepo fetch --cflags` prints paths into `~/.xmake/packages/...` — fine for local builds, not portable across machines. Use `xrepo export` to get a self-contained directory.

## When to branch out

- Virtual environments (`xrepo env`) → `xrepo-env`
- Using packages from inside an `xmake.lua` project → `xmake-packages`
- Writing / testing a package recipe → `xmake-repo-testing`
- Cross-toolchain setup used by `-p/-a` → `xmake-toolchains`
