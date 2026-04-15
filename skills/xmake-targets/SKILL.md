---
name: xmake-targets
description: Use when configuring targets in xmake.lua — setting kind (binary/static/shared), adding files/includes/links, defining dependencies between targets, and applying per-target compile/link flags.
---

# Xmake Targets

A target is a unit of build output. Every buildable artifact lives inside `target("name") ... target_end()` (or just the indented block until the next top-level call).

## Kinds

```lua
target("app")
    set_kind("binary")     -- executable
target("mylib")
    set_kind("static")     -- .a / .lib
target("mydyn")
    set_kind("shared")     -- .so / .dll / .dylib
target("headeronly")
    set_kind("headeronly") -- no compilation, just install headers
```

## Sources and headers

```lua
target("foo")
    set_kind("static")
    add_files("src/*.cpp", "src/**/*.cpp")
    remove_files("src/legacy/*.cpp")
    add_includedirs("include", {public = true})   -- public = also exposed to dependents
    add_headerfiles("include/(**.h)")              -- for `xmake install`
```

Globs: `*` matches one level, `**` matches recursively. `(...)` in `add_headerfiles` preserves directory structure.

## Dependencies between targets

```lua
target("mylib")
    set_kind("static")
    add_files("src/lib/*.cpp")
    add_includedirs("include", {public = true})

target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_deps("mylib")      -- inherits public includes/defines/links
```

## Flags and defines

```lua
target("app")
    set_languages("c++17")
    set_warnings("all", "error")
    set_optimize("fastest")      -- none|faster|fastest|smallest
    set_symbols("debug")         -- debug|hidden
    add_defines("MY_MACRO=1", {public = true})
    add_cxflags("-Wshadow", {force = true})
    add_ldflags("-Wl,--as-needed")
    add_links("pthread")         -- prefer add_syslinks on cross-platform code
    add_syslinks("pthread", "dl")
```

`{public = true}` propagates to dependents; `{force = true}` skips the automatic flag-detection filter.

## Target-level files by platform/mode

```lua
target("app")
    if is_plat("windows") then
        add_files("src/win/*.cpp")
    elseif is_plat("linux", "macosx") then
        add_files("src/posix/*.cpp")
    end
    if is_mode("debug") then
        add_defines("DEBUG")
    end
```

## Before/after hooks

```lua
target("app")
    after_build(function (target)
        print("built: %s", target:targetfile())
    end)
```

Hooks available: `on_load`, `before_build`, `on_build`, `after_build`, `before_install`, `after_install`, `before_run`, `after_run`, etc.

## When to branch out

- User options gating these settings → `xmake-options`
- Linking a 3rd-party library → `xmake-packages`
- Reusable config across targets → `xmake-rules`
