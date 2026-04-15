---
name: xmake-nim
description: Use when building Nim projects with xmake — binary/library targets with `.nim` sources, choosing the C backend (the default) vs C++, and mixing Nim with C.
---

# Building Nim with Xmake

Nim compiles to C (or C++ / JavaScript) via `nim c` / `nim cpp`; xmake wraps this into a regular target model.

## 1. Minimal project

```bash
xmake create -l nim -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.nim")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. Backend selection

```lua
target("app")
    add_files("src/main.nim")
    set_values("nim.backend", "c")         -- default
    -- or:
    set_values("nim.backend", "cpp")
    set_values("nim.backend", "objc")
    set_values("nim.backend", "js")
```

Most projects use `c` (native). `cpp` is required if you import C++ libraries via `{.importcpp.}`. `js` is for web targets.

## 4. Flags

```lua
target("app")
    add_files("src/*.nim")
    add_ncflags("-d:release", "--opt:speed")           -- Nim compiler flags
    add_cxflags("-O3")                                 -- passes to underlying C compiler
```

`add_ncflags` = Nim compiler flags. `add_cxflags` flows through to the C backend.

## 5. Toolchain

```bash
xmake f --toolchain=nim
xmake f --toolchain=nim --sdk=/opt/nim-2.0
```

## 6. Mixing Nim with C

Nim code can call C with `{.importc.}`; xmake can also compile C files alongside:

```lua
target("app")
    set_kind("binary")
    add_files("src/main.nim", "src/helper.c")
    add_includedirs("include")
```

## Pitfalls

- **Backend mismatch.** If Nim code uses `{.importcpp.}`, the target must use `nim.backend = "cpp"`. Otherwise link errors.
- **`nimble` vs xmake.** Nim's native package manager is `nimble`. Xmake doesn't integrate with it — declare Nim deps via nimble manually, and let xmake handle the build.
- **`-d:release` passed as nim flag.** It's a Nim define, not a C flag — must go in `add_ncflags`, not `add_defines`.

## When to branch out

- Target basics → `xmake-targets`
- Mixing with C/C++ → `xmake-targets`
