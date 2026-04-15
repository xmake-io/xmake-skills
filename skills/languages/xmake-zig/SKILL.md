---
name: xmake-zig
description: Use when building Zig projects with xmake — binary/library targets with `.zig` sources, cross-compilation (one of Zig's strengths), and mixing Zig with C.
---

# Building Zig with Xmake

Xmake drives `zig build-exe` / `zig build-lib` under the hood. Zig is particularly nice because its native cross-compile story plays well with xmake's toolchain selection.

## 1. Minimal project

```bash
xmake create -l zig -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.zig")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. Toolchain

```bash
xmake f --toolchain=zig
xmake f --toolchain=zig --sdk=/opt/zig-0.12
```

Zig is auto-detected when `zig` is on PATH. Pin with `--sdk` for a specific install.

## 4. Cross-compile

Zig has built-in cross-compile for every target it supports — no extra SDK needed:

```bash
xmake f -p windows -a x86_64 --toolchain=zig
xmake f -p linux   -a arm64  --toolchain=zig
xmake f -p macosx  -a arm64  --toolchain=zig
xmake
```

Xmake passes `-target <triple>` to `zig build-exe`. Far easier than GCC/Clang cross-SDKs.

## 5. Mixing Zig with C

Zig ships its own C compiler (`zig cc`) that can build C files. You can mix them in one target:

```lua
target("app")
    set_kind("binary")
    add_files("src/main.zig", "src/helper.c")
    add_includedirs("include")
```

Zig imports C via `@cImport`. No bridging needed.

## 6. Using Zig as your C compiler

Bonus trick: point xmake's C toolchain at `zig cc` for free cross-compile of C projects:

```bash
xmake f --toolchain=zig
xmake f --cc="zig cc" --cxx="zig c++"
```

This is especially useful for MinGW-free Windows builds from Linux.

## Pitfalls

- **Zig API churn.** Zig is pre-1.0; `build-exe` / `build-lib` flags change between versions. Pin `--sdk` to a known version in CI.
- **Mixing `.zig` and `.cpp` in one target.** `.zig` + `.c` is fine; C++ is more brittle — use a separate target with `add_deps`.
- **Runtime dependencies.** `-target <triple>` gives you a cross-compile, but you still need the right dynamic linker at runtime on the target — Zig can produce static binaries if you prefer (`-fno-pie` etc.).

## When to branch out

- Cross-compile plumbing (`-p`, `-a`, `--sdk`) → `xmake-cross-compilation`
- Target basics → `xmake-targets`
