---
name: xmake-toolchains
description: Use when switching compilers (gcc/clang/msvc/mingw), setting up cross-compilation for Android/iOS/embedded/wasm, or defining a custom toolchain.
---

# Xmake Toolchains

A toolchain bundles a compiler, linker, archiver, and related tools. You pick one per configure.

## Listing and switching

```bash
xmake show -l toolchains             # list available toolchains
xmake f --toolchain=clang
xmake f --toolchain=gcc-11
xmake f --toolchain=mingw --mingw=/opt/llvm-mingw
xmake f --toolchain=msvc --vs=2022
```

In `xmake.lua`:

```lua
set_toolchains("clang")

target("app")
    set_toolchains("gcc-11")          -- override per target
```

## Cross-compilation

### Android

```bash
xmake f -p android --ndk=~/android-ndk-r26 -a arm64-v8a
xmake
```

Arch options: `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`.

### iOS / macOS

```bash
xmake f -p iphoneos -a arm64
xmake f -p macosx -a arm64        # Apple Silicon
```

### Windows from Linux (mingw)

```bash
xmake f -p mingw --mingw=/usr/x86_64-w64-mingw32 -a x86_64
```

### WebAssembly

```bash
xmake f -p wasm --toolchain=emcc
```

### Generic cross-gcc

```bash
xmake f -p cross --sdk=/opt/my-sdk --cross=arm-linux-gnueabihf- -a arm
```

`--sdk` points to the SDK root; `--cross` is the tool prefix.

## Defining a custom toolchain

```lua
toolchain("my-cross")
    set_kind("standalone")
    set_sdkdir("/opt/my-sdk")
    set_toolset("cc",  "arm-linux-gnueabihf-gcc")
    set_toolset("cxx", "arm-linux-gnueabihf-g++")
    set_toolset("ld",  "arm-linux-gnueabihf-g++")
    set_toolset("ar",  "arm-linux-gnueabihf-ar")
    set_toolset("strip", "arm-linux-gnueabihf-strip")
toolchain_end()
```

```bash
xmake f --toolchain=my-cross -p cross -a arm
```

## Using a toolchain only for one target

```lua
target("firmware")
    set_toolchains("my-cross")
    set_plat("cross")
    set_arch("arm")
```

## Platform / mode probes

```lua
if is_plat("android", "iphoneos") then ... end
if is_arch("arm64", "arm64-v8a") then ... end
if is_host("windows") then ... end         -- the machine running xmake
```

## When to branch out

- Adding packages that also need cross-build → `xmake-packages`
- Target-level compile flags → `xmake-targets`
