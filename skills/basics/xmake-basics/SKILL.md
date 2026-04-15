---
name: xmake-basics
description: Use when the user is starting a new Xmake project, installing Xmake, or needs the minimal xmake.lua structure. Covers installation, `xmake create`, the project layout, and the smallest valid xmake.lua.
---

# Xmake Basics

Xmake is a cross-platform build utility based on Lua. A project is driven by a single `xmake.lua` file at the repository root.

## Installation

- macOS: `brew install xmake`
- Linux: `curl -fsSL https://xmake.io/shget.text | bash`
- Windows: download the installer from https://xmake.io or `scoop install xmake`

Verify with `xmake --version`.

## Creating a project

```bash
xmake create -l c++ -P hello        # C++ console app
xmake create -l c -t static hello   # C static library
```

Templates: `console` (default), `static`, `shared`, `binary`. List with `xmake create --list`.

## Minimal xmake.lua

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.cpp")
```

That is enough to build with `xmake` and run with `xmake run hello`.

## Build modes

`mode.debug` / `mode.release` are built-in rules. Switch with:

```bash
xmake f -m debug     # or release
xmake
```

## Typical layout

```
myproject/
  xmake.lua
  src/
    main.cpp
  include/
  tests/
```

`add_files("src/*.cpp")` globs source files; `add_includedirs("include")` exposes headers.

## When to branch out

- Target options (kind, deps, flags) → see `xmake-targets`
- Adding a 3rd-party library → see `xmake-packages`
- User-configurable options → see `xmake-options`
- Cross-compilation / alternative compilers → see `xmake-toolchains`
