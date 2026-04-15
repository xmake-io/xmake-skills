---
name: xmake-dlang
description: Use when building D projects with xmake — binary/library targets with `.d` sources, DUB package integration via `add_requires("dub::...")`, and selecting DMD / LDC / GDC compilers.
---

# Building D (dlang) with Xmake

D has three main compilers (DMD, LDC, GDC); xmake drives all of them through a unified `target()` API and supports DUB for package dependencies.

## 1. Minimal project

```bash
xmake create -l dlang -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.d")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. DUB dependencies (v2.3.6+)

```lua
add_requires("dub::vibe-d ~>0.9.5")
add_requires("dub::libasync")

target("app")
    set_kind("binary")
    add_files("src/*.d")
    add_packages("dub::vibe-d", "dub::libasync")
```

Known limitation: cascading transitive deps must be declared manually for now. Add every dep in the tree explicitly.

## 4. Compiler selection

```bash
xmake f --toolchain=dlang                    # DMD (default)
xmake f --toolchain=ldc
xmake f --toolchain=gdc
```

Or pin paths:

```bash
xmake f --toolchain=ldc --sdk=/opt/ldc-1.30
```

LDC is usually the best choice for release builds (LLVM backend, best optimization). DMD is fastest to compile with.

## 5. Flags

```lua
target("app")
    add_files("src/*.d")
    add_dcflags("-O", "-release")          -- passed to the D compiler
    add_ldflags("-Wl,--as-needed")
```

`add_dcflags` = D compiler flags. Each compiler has its own flag dialect — prefer portable ones or gate with `if is_toolchain("ldc") then ... end`.

## 6. Mixing D with C

```lua
target("app")
    set_kind("binary")
    add_files("src/main.d", "src/extern.c")    -- mix C and D in one target
    add_includedirs("include")
```

D can call C directly via `extern (C)` declarations on the D side. No bridging needed.

## Pitfalls

- **Transitive DUB deps.** You must add them manually until xmake's DUB integration is complete.
- **Compiler-specific flags leaking across toolchains.** `add_dcflags("-mattr=+avx2")` works on LDC, fails on DMD. Gate with `is_toolchain`.
- **D shared libraries on Windows.** DMD and LDC produce different import-library formats; stick to one per project on Windows.

## When to branch out

- Build modes → `xmake-targets`, `xmake-build-optimization`
- Cross-compile → `xmake-cross-compilation` (LDC supports many targets)
- Package management basics → `xmake-packages`
