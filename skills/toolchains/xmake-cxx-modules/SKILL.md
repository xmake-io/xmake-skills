---
name: xmake-cxx-modules
description: Use when building C++20 modules with xmake — setting up `.mpp`/`.cppm`/`.ixx` sources, enabling `build.c++.modules`, configuring per-compiler quirks (clang/gcc/msvc), `moduleonly` libraries, and distributing module packages. Covers typical project layouts for executable/library/headeronly/moduleonly targets.
---

# Building C++20 Modules with Xmake

Xmake has first-class C++20 modules support across GCC, Clang, and MSVC. It handles the BMI (Built Module Interface) cache, dependency scanning, and per-compiler flag differences automatically — you mostly just point it at the files and enable the policy.

## 1. Prerequisites

- **C++20** language version.
- **A new-ish compiler**: GCC 14+, Clang 16+ (better: 17+), MSVC 19.34+ (VS 2022 17.4+).
- **`build.c++.modules` policy enabled.**

## 2. Minimal executable with modules

Project layout:

```
myapp/
├── xmake.lua
└── src/
    ├── main.cpp             # importer (regular .cpp)
    └── math.mpp             # module interface
```

```cpp
// src/math.mpp
export module math;

export int add(int a, int b) {
    return a + b;
}
```

```cpp
// src/main.cpp
import math;
#include <iostream>

int main() {
    std::cout << add(2, 3) << "\n";
    return 0;
}
```

```lua
-- xmake.lua
add_rules("mode.debug", "mode.release")

target("myapp")
    set_kind("binary")
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_files("src/main.cpp", "src/math.mpp")
```

Build + run:

```bash
xmake f -m release
xmake
xmake run myapp
```

That's it — xmake scans the `.mpp`, builds the BMI, and links.

## 3. File extension conventions

Xmake recognizes all common module interface extensions:

| Extension | Used by | Notes |
| --- | --- | --- |
| `.mpp` | xmake convention | Recommended — cross-compiler |
| `.cppm` | Clang convention | Works everywhere with xmake |
| `.ixx` | MSVC convention | Works everywhere with xmake |
| `.mxx` | Alternative | Also recognized |
| `.cpp` | Regular source | Only a module if it starts with `module;` or `export module` |

Use whichever you prefer; xmake treats them uniformly. `.mpp` is the xmake default in examples.

## 4. Module target kinds

### Binary (executable)

```lua
target("app")
    set_kind("binary")
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_files("src/*.cpp", "src/*.mpp")
```

### Static/shared library exposing modules

```lua
target("mylib")
    set_kind("static")              -- or "shared"
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_files("src/*.cpp", "src/*.mpp")
```

Consumers that `add_deps("mylib")` inherit the module search path automatically.

### `moduleonly` — pure module library

When a library has **no non-module sources** (no `.cpp`, only `.mpp`/`.cppm`), use `moduleonly`. Xmake skips the linker and just ships the BMI + sources.

```lua
target("math_modules")
    set_kind("moduleonly")
    set_languages("c++20")
    add_files("src/*.mpp")
```

This is the right choice for header-only-style module libraries. Consumers still build the BMI but don't link anything.

### Headeronly with module hints

For mixed header/module libs, keep `headeronly` for the header side and a separate `moduleonly` target for the module side:

```lua
target("mylib_headers")
    set_kind("headeronly")
    add_headerfiles("include/(**.h)")

target("mylib_modules")
    set_kind("moduleonly")
    set_languages("c++20")
    add_files("src/*.mpp")
```

## 5. Module dependencies between files

Xmake scans `import ...;` statements and orders compilation automatically. No manual dep lists needed:

```cpp
// src/math.mpp
export module math;
export int add(int a, int b) { return a + b; }
```

```cpp
// src/geometry.mpp
export module geometry;
import math;                    // xmake sees this, builds math first
export int perimeter(int w, int h) { return 2 * (w + h); }
```

```lua
target("app")
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_files("src/main.cpp", "src/math.mpp", "src/geometry.mpp")
```

## 6. Module partitions

```cpp
// src/math/core.mpp
export module math:core;
export int add(int a, int b) { return a + b; }
```

```cpp
// src/math/vec.mpp
export module math:vec;
export struct Vec { int x, y; };
```

```cpp
// src/math.mpp (primary interface)
export module math;
export import :core;
export import :vec;
```

Just add them all to `add_files` — xmake handles partition ordering.

## 7. Standard library modules

`import std;` / `import std.compat;` — supported where the compiler supports it:

- **Clang 17+**: native `import std;` once libc++ modules are built.
- **MSVC 19.35+ (VS 2022 17.5+)**: native `import std;` out of the box.
- **GCC 14+**: partial; check the release notes.

Enable in xmake:

```lua
target("app")
    set_languages("c++23")          -- std modules usually need c++23
    set_policy("build.c++.modules", true)
    set_policy("build.c++.modules.std", true)
    add_files("src/*.cpp", "src/*.mpp")
```

```cpp
// src/main.cpp
import std;
int main() { std::cout << "hi\n"; }
```

## 8. Per-compiler notes

### Clang

- Best cross-platform support. Use Clang 17+ for production.
- Needs `libc++` for `import std;`.
- Xmake passes `-fmodules -fbuiltin-module-map` as needed.

### GCC

- GCC 14+ is the realistic baseline. GCC 11–13 have partial/buggy support.
- Dependency scanner is less mature than Clang's — if a file is mis-ordered, `xmake -rv` and inspect.

### MSVC

- Works since VS 2022 17.4, best on 17.6+.
- `.ixx` is MSVC's natural extension but `.mpp`/`.cppm` also work.
- Incremental builds on MSVC are slower than Clang.

### Mixed toolchains

Stick with one toolchain per build. Mixing Clang BMIs with GCC/MSVC won't work — BMI format is compiler-specific.

## 9. Dependency packages

Consuming a module package:

```lua
add_requires("fmt")

target("app")
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_files("src/main.cpp")
    add_packages("fmt")
```

For a package that *is* modules-only, see `xmake-private-packages`:

```lua
package("foo")
    set_kind("library", {moduleonly = true})
    set_sourcedir(path.join(os.scriptdir(), "src"))
    on_install(function (package)
        import("package.tools.xmake").install(package, {})
    end)
```

## 10. Inspecting the build

```bash
xmake -v             # show actual compile commands (useful for BMI flags)
xmake -rv            # verbose rebuild
xmake clean --all    # drop BMI cache when debugging dep-order bugs
```

BMI files live under `build/.gens/<target>/`. Safe to delete — xmake will regenerate.

## Common pitfalls

- **Forgot `set_policy("build.c++.modules", true)`.** Xmake then builds `.mpp` as regular C++ and ignores `import`. Symptom: "unresolved external" or `import` treated as an error.
- **`set_languages("c++17")`.** Modules require `c++20` (or `c++23` for std modules).
- **Compiler too old.** "imports are not enabled" → upgrade. Xmake can't work around missing compiler support.
- **Stale BMI after edit.** Dep-scan sometimes misses an edit; `xmake -rv` or `xmake clean --all`.
- **Mixing toolchains between dependent targets.** `mylib` built with Clang, `app` with GCC → BMI is unreadable. Rebuild all targets with one toolchain.
- **`moduleonly` with regular sources.** Xmake refuses to link — either split the `.cpp` into its own `static`/`shared` target, or drop `moduleonly` and use `static`.
- **`import std;` on a compiler that doesn't support it.** Even with the policy enabled, the compiler must ship libc++/MSVC std modules. Fall back to `#include <iostream>` if unsupported.
- **Module cycles.** `a.mpp` imports `b`, `b.mpp` imports `a`. C++20 modules forbid cycles — refactor one file into a partition or restructure.

## When to branch out

- Target kinds (binary/static/shared/headeronly/moduleonly) → `xmake-targets`
- Compiler / toolchain selection → `xmake-toolchains`, `xmake-cross-compilation`
- Distributing a module library → `xmake-private-packages`
- Slow module builds → `xmake-build-optimization`
