---
name: xmake-pascal
description: Use when building Pascal projects with xmake — `.pas`/`.pp`/`.lpr` sources, FreePascal (`fpc`) / Delphi toolchain, binary and shared-library targets.
---

# Building Pascal with Xmake

Xmake supports Pascal (via FreePascal `fpc`) since v2.5.8.

## 1. Minimal project

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/main.pas")
```

Recognized extensions: `.pas`, `.pp`, `.lpr`.

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("shared")       -- .so / .dll
target("staticlib") set_kind("static")       -- .a / .lib
```

Shared library example:

```lua
target("mylib")
    set_kind("shared")
    add_files("src/lib.pas")
```

## 3. Flags

```lua
target("app")
    add_files("src/*.pas")
    add_pcflags("-O3", "-Xs")            -- fpc flags (strip symbols)
```

## 4. Toolchain

```bash
xmake f --toolchain=fpc                   # FreePascal
xmake f --toolchain=fpc --sdk=/opt/fpc-3.2
```

## Pitfalls

- **Project file format.** `.lpr` is the standard program entry for Lazarus projects — use it as the `add_files` entry, not unit files.
- **Unit dependency ordering.** `fpc` handles unit deps itself; don't enumerate dependencies manually unless you have a reason.
- **Cross-compile.** FreePascal supports many targets but the config is fpc-specific; see FreePascal docs for `-T<target>` / `-P<arch>`.

## When to branch out

- Target basics → `xmake-targets`
- Cross-compile → `xmake-cross-compilation`
