---
name: xmake-rust
description: Use when building Rust projects with xmake — binary/library targets with `.rs` sources, Cargo package integration via `add_requires("cargo::...")` or full `Cargo.toml` management, C↔Rust interop with `cxxbridge`, `rustc` edition / toolchain selection, and cross-compile.
---

# Building Rust with Xmake

Xmake builds Rust projects via `rustc` (and optionally cargo for dependency resolution). It gives you the same declarative `target()` / `add_files()` model you use for C/C++, plus special support for Cargo package integration and C↔Rust FFI.

## 1. Minimal project

```bash
xmake create -l rust -t console hello
cd hello
xmake run
```

Generated `xmake.lua`:

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/main.rs")
```

That's it — same shape as C++, just `.rs` files.

## 2. Target kinds

```lua
target("app")       set_kind("binary")      -- executable
target("mylib")     set_kind("static")      -- .rlib / .a
target("mydyn")     set_kind("shared")      -- cdylib
```

## 3. Editions / flags

```lua
target("app")
    set_kind("binary")
    add_files("src/main.rs")
    set_values("rust.edition", "2021")       -- or "2018", "2015", "2024"
    add_rcflags("-C", "opt-level=3", {force = true})
```

`add_rcflags` is the Rust equivalent of `add_cxflags`. `{force = true}` bypasses the flag auto-filter.

## 4. Cargo dependencies — the quick way

For a handful of crates, use `cargo::name version` in `add_requires`:

```lua
add_requires("cargo::base64 0.13.0")
add_requires("cargo::serde 1.0", {configs = {features = {"derive"}}})

target("app")
    set_kind("binary")
    add_files("src/main.rs")
    add_packages("cargo::base64", "cargo::serde")
```

Downside: if two of your direct deps share a transitive dep at different versions, you hit "multiple versions of crate" errors. For non-trivial dep trees, use the full-Cargo.toml path below.

## 5. Cargo dependencies — the full `Cargo.toml` way

Write a normal `Cargo.toml` next to your `xmake.lua`, then let xmake read it:

```lua
add_requires("cargo::deps", {configs = {cargo_toml = path.join(os.projectdir(), "Cargo.toml")}})

target("app")
    set_kind("binary")
    add_files("src/main.rs")
    add_packages("cargo::deps")
```

Cargo resolves the full graph (including transitives) once, xmake consumes the result. Use this for any project with more than a couple of crates.

## 6. Rust ↔ C++ interop

### Call Rust from C++ (`cxxbridge`)

```lua
add_requires("cargo::cxx 1.0")

target("bridge")
    set_kind("static")
    add_files("src/lib.rs")
    add_packages("cargo::cxx")

target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_deps("bridge")
```

Use `#[cxx::bridge]` in `src/lib.rs` as with a normal cxx project. Xmake wires up the header generation.

### Call C++ from Rust

```lua
target("cxxlib")
    set_kind("static")
    add_files("src/cxx/*.cpp")

target("app")
    set_kind("binary")
    add_files("src/main.rs")
    add_deps("cxxlib")
```

Rust links `cxxlib` as a native static library via `-l`. Use `extern "C"` on the C++ side, `extern "C" { ... }` block on the Rust side.

Full examples: [`tests/projects/rust/cxx_call_rust_library`](https://github.com/xmake-io/xmake/tree/master/tests/projects/rust/cxx_call_rust_library) and [`rust_call_cxx_library`](https://github.com/xmake-io/xmake/tree/master/tests/projects/rust/rust_call_cxx_library).

## 7. Toolchain

```bash
xmake f --toolchain=rust                       # force rustc as the toolchain
xmake show -l toolchains                       # see what else is available
```

Xmake auto-detects `rustc` on PATH. Pin a specific install via `--sdk`:

```bash
xmake f --toolchain=rust --sdk=$HOME/.rustup/toolchains/stable-x86_64-apple-darwin
```

## 8. Cross-compile

```bash
xmake f -p windows -a x86_64 --toolchain=rust
xmake
```

Under the hood xmake passes `--target=x86_64-pc-windows-msvc` (or equivalent) to `rustc`. The corresponding Rust target component must be installed:

```bash
rustup target add x86_64-pc-windows-msvc
```

## 9. Running tests

```bash
xmake run app --some-arg
```

Or add test cases:

```lua
target("app_test")
    set_kind("binary")
    add_files("tests/*.rs")
    add_tests("default")
```

See `xmake-tests`.

## Pitfalls

- **Multiple crate versions.** `cargo::foo` + transitive `cargo::foo` at a different version → link errors. Switch to the full `Cargo.toml` route.
- **Missing `rustup target add`.** Cross-compile fails with "can't find crate for `std`". Install the Rust target component first.
- **`set_languages("...")`.** Doesn't apply to Rust — use `set_values("rust.edition", "2021")`.
- **Mixing `cargo::` and manual `-l` flags.** Let `add_packages("cargo::...")` own the link line; mixing hand-rolled `add_links` gets messy.
- **`.rs` and `.cpp` in the same target.** Not supported — split into two targets with `add_deps`.

## When to branch out

- Build modes / debug vs release → `xmake-targets`, `xmake-build-optimization`
- Cross-compile plumbing → `xmake-cross-compilation`
- Package management concepts → `xmake-packages`
- Testing → `xmake-tests`
