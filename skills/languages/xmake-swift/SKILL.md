---
name: xmake-swift
description: Use when building Swift projects with xmake — binary/library targets with `.swift` sources, Swift ↔ C++/Objective-C interop (`swift.interop` value), Swift module name configuration, and iOS/macOS builds.
---

# Building Swift with Xmake

Xmake builds Swift through `swiftc` and integrates with Xcode on Apple platforms. Use it when you want the same `xmake.lua` to drive a project that mixes Swift with C/C++/Objective-C.

## 1. Minimal project

```bash
xmake create -l swift -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.swift")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. Swift ↔ C++/Objective-C interop (v3.0.5+)

Xmake supports automatic Swift-interop C++ header generation through the `swift.interop` target value.

### Call Swift from C++

```lua
target("myswift")
    set_kind("static")
    add_files("src/*.swift")
    set_values("swift.interop", "cxx")                 -- or "objc"
    set_values("swift.modulename", "MySwift")          -- C++ namespace name

target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_deps("myswift")
    set_languages("c++17")
```

Xmake generates a C++ header with `#include "MySwift-Swift.h"` style, exposing Swift types under `MySwift::` namespace.

### Interop modes

- `"objc"` — Objective-C interop (classic, works since early Swift)
- `"cxx"` — C++ interop (Swift 5.9+ required)

### Resolving duplicate `main`

When both Swift and C++ have a `main`, Swift's usually wins. Force the C++ one:

```lua
set_values("swift.interop.cxxmain", true)
```

## 4. Apple platforms

```bash
xmake f -p macosx  -a arm64
xmake f -p iphoneos -a arm64
xmake f -p iphonesimulator -a arm64
xmake
```

Xmake picks up Xcode automatically. Pin SDK version with `--xcode_sdkver=16.2`, deployment target with `--target_minver=12.0`.

## 5. Flags

```lua
target("app")
    add_files("src/*.swift")
    add_scflags("-O")                  -- passed to swiftc
    add_ldflags("-Wl,-dead_strip")
```

`add_scflags` = Swift compiler flags.

## 6. Toolchain

```bash
xmake f --toolchain=swift
xmake f --toolchain=swift --sdk=/Applications/Xcode.app/Contents/Developer
```

On Linux, install swift from swift.org and point `--sdk` at the install root.

## Pitfalls

- **C++ interop needs Swift 5.9+.** Older Xcode won't compile `swift.interop = "cxx"` projects.
- **Swift modules are per-target.** Each `target("...")` that has `.swift` files produces a module; don't split Swift sources across unrelated targets without thinking about the module boundary.
- **Forgetting `set_values("swift.modulename", ...)`.** Interop uses the module name as the C++ namespace — default is the target name, and if that has hyphens you'll get invalid C++.
- **Mixing Swift and Objective-C in one target without `swift.interop = "objc"`.** Add a bridging header manually, or use the interop value.

## When to branch out

- Apple-platform cross-compile (iOS, tvOS, watchOS) → `xmake-cross-compilation`
- Target basics → `xmake-targets`
- C++ side configuration → `xmake-cxx-modules` (if using C++20 modules)
