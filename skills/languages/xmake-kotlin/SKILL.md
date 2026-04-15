---
name: xmake-kotlin
description: Use when building Kotlin/Native projects with xmake — `.kt` sources for native (not JVM) targets, using the `kotlin-native` toolchain, producing executables and static/shared libraries.
---

# Building Kotlin (Native) with Xmake

Xmake's Kotlin support targets **Kotlin/Native** — the ahead-of-time-compiled flavor, not JVM. For JVM-side projects, use Gradle/Maven directly.

## 1. Minimal project

```bash
xmake create -l kotlin -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.kt")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")       -- Kotlin/Native executable
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. Toolchain

Kotlin/Native must be installed separately — it's not bundled with xmake.

```bash
xmake f --toolchain=kotlin-native
xmake f --toolchain=kotlin-native --sdk=/opt/kotlin-native-prebuilt-linux-x86_64-1.9.0
```

Download the compiler from [kotlinlang.org/docs/native-overview.html](https://kotlinlang.org/docs/native-overview.html) and point `--sdk` at it.

## 4. Targets

Kotlin/Native supports many platforms:

```bash
xmake f -p linux   -a x86_64 --toolchain=kotlin-native
xmake f -p macosx  -a arm64  --toolchain=kotlin-native
xmake f -p windows -a x86_64 --toolchain=kotlin-native
xmake f -p iphoneos -a arm64 --toolchain=kotlin-native
```

## 5. Interop with C

Kotlin/Native has a `cinterop` mechanism for binding C libraries. At the xmake level, link native static libs the usual way:

```lua
target("app")
    set_kind("binary")
    add_files("src/*.kt")
    add_deps("mycdep")        -- C static library target
```

`cinterop` configuration (the `.def` files) is Kotlin-side — see Kotlin/Native docs.

## Pitfalls

- **Use case is narrow.** Most real-world Kotlin is JVM / Android. If that's what you need, Gradle is the right tool, not xmake.
- **Large toolchain download.** Kotlin/Native is ~1GB. Cache `--sdk` globally via `xmake g --sdk=...`.
- **Slow first compile.** Kotlin/Native's cold start is notoriously slow. Expect longer builds than Rust/Go for equivalent code.

## When to branch out

- Target basics → `xmake-targets`
- Cross-compile → `xmake-cross-compilation`
