---
name: xmake-vala
description: Use when building Vala projects with xmake — `.vala` sources with the `add_rules("vala")` rule, glib dependency, declaring used VAPI packages with `add_values("vala.packages", ...)`, and producing executables or shared/static libraries with exported headers/VAPIs.
---

# Building Vala with Xmake

Vala compiles to C (via `valac`), then to native code. Xmake handles both steps via the `vala` rule. Supported since v2.5.7 (binaries) / v2.5.8 (libraries).

## 1. Minimal console program

```lua
add_rules("mode.debug", "mode.release")
add_requires("glib")                                 -- mandatory

target("hello")
    set_kind("binary")
    add_rules("vala")                                -- required
    add_files("src/*.vala")
    add_packages("glib")
    add_values("vala.packages", "glib-2.0", "gobject-2.0")
```

Three things every Vala target needs:

- `add_rules("vala")` — tells xmake this is a Vala target.
- `add_requires("glib")` + `add_packages("glib")` — Vala's runtime.
- `add_values("vala.packages", ...)` — the VAPI package names that `valac` should import.

## 2. Static library with exported API

```lua
target("mymath")
    set_kind("static")
    add_rules("vala")
    add_files("src/*.vala")
    add_packages("glib")
    add_values("vala.packages", "glib-2.0", "gobject-2.0")
    add_values("vala.header", "mymath.h")             -- exported C header
    add_values("vala.vapi",   "mymath-1.0.vapi")      -- exported VAPI
```

Consumers in C can `#include "mymath.h"`; consumers in Vala import `mymath-1.0.vapi`.

## 3. Shared library

```lua
target("mymath")
    set_kind("shared")
    add_rules("vala")
    add_files("src/*.vala")
    add_packages("glib")
    add_values("vala.packages", "glib-2.0", "gobject-2.0")
    add_values("vala.header", "mymath.h")
    add_values("vala.vapi",   "mymath-1.0.vapi")
```

## 4. Consuming a Vala library in C

```lua
target("app")
    set_kind("binary")
    add_files("src/main.c")
    add_deps("mymath")
    add_packages("glib")
```

The exported header and VAPI flow through `add_deps` automatically.

## Pitfalls

- **Forgot `add_rules("vala")`.** `.vala` files are treated as unknown — build fails. Always add the rule.
- **Missing glib dep.** Vala's runtime is glib. Add it as a package even for "hello world".
- **`vala.packages` vs `add_packages`.** These are different axes: `vala.packages` is what `valac` imports (VAPI names), `add_packages` is what xmake links. You usually need both for glib.
- **Version in the VAPI name.** `glib-2.0`, not `glib`. Check `pkg-config --list-all` for the exact names.
- **Cross-compile.** Vala's two-step build (Vala → C → native) makes cross-compile trickier than pure C. Works but requires the cross toolchain to also have glib headers.

## When to branch out

- Package management → `xmake-packages`
- Target basics → `xmake-targets`
