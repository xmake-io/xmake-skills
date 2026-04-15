---
name: xmake-options
description: Use when defining user-configurable build options (`option(...)`), reading them from targets, or exposing them on the `xmake f` command line. Covers showmenu, default, values, and option-driven config.h generation.
---

# Xmake Options

Options are user-facing switches that can be set via `xmake f --name=value` and queried from targets and conditions.

## Declaring an option

```lua
option("enable_foo")
    set_default(false)
    set_showmenu(true)
    set_description("Enable the foo subsystem")
option_end()
```

`set_showmenu(true)` makes the option visible in `xmake f --menu` and `xmake f --help`.

## Using an option from a target

```lua
target("app")
    add_files("src/*.cpp")
    if has_config("enable_foo") then
        add_files("src/foo/*.cpp")
        add_defines("ENABLE_FOO=1")
    end
```

`has_config("name")` returns true when the option is truthy. Use `get_config("name")` to read the actual value.

## Enumerated / typed options

```lua
option("backend")
    set_default("vulkan")
    set_values("vulkan", "metal", "d3d12")
    set_showmenu(true)
```

```bash
xmake f --backend=metal
```

## Options that probe the host

```lua
option("has_avx2")
    add_cxflags("-mavx2")
    add_cxxincludes("immintrin.h")
    add_cxxsnippets("int f(){ return _mm256_set1_epi32(1)[0]; }")
```

Xmake runs the snippet at configure time; the option becomes enabled if the probe compiles.

## Generating a config header

```lua
target("app")
    set_configdir("$(buildir)/config")
    add_configfiles("src/config.h.in")
    set_configvar("ENABLE_FOO", has_config("enable_foo") and 1 or 0)
```

`src/config.h.in` can reference `${define ENABLE_FOO}` or `${ENABLE_FOO}`.

## Command-line usage

```bash
xmake f --enable_foo=y --backend=metal -m release
xmake
```

## When to branch out

- Per-target flags — `xmake-targets`
- Toolchain selection via options — `xmake-toolchains`
