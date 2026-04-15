---
name: xmake-cross-compilation
description: Use when the user is cross-compiling a C/C++ project with xmake — targeting Android/iOS/embedded/MinGW/WASM, pointing at an SDK/NDK root, overriding cc/cxx/ld/ar, selecting a built-in cross toolchain, or defining a custom toolchain in xmake.lua. Covers the full `xmake f` flag set and common SDK layouts. For generic toolchain switching on the *host* platform (e.g. gcc ↔ clang), `xmake-toolchains` is enough.
---

# Cross Compilation with Xmake

Cross compilation means producing binaries for a platform that is **not** the host you are running xmake on. In xmake it boils down to three choices:

1. Which **platform** (`-p/--plat=`) you are targeting.
2. Which **toolchain** (SDK + cross prefix + cc/cxx/ld/ar) does the work.
3. Any **extra flags** (includes, libs, cflags) needed for the target.

Most of the time, `-p <plat> --sdk=<root>` is enough — xmake auto-detects the rest. The rest of this skill is the knobs you reach for when auto-detection is not enough.

## 1. Typical cross-SDK layout

Xmake auto-detects a toolchain SDK that looks like this:

```
/home/toolchains_sdkdir
├── bin/
│   ├── arm-linux-gnueabihf-gcc
│   ├── arm-linux-gnueabihf-g++
│   ├── arm-linux-gnueabihf-ld
│   ├── arm-linux-gnueabihf-ar
│   └── arm-linux-gnueabihf-strip
├── lib/
│   └── libc.so, libstdc++.so, ...
└── include/
    └── stdio.h, ...
```

The `arm-linux-gnueabihf-` part is the **cross prefix**. Give xmake just `--sdk=/home/toolchains_sdkdir` and it will:

- Find the `bin/` dir.
- Detect `arm-linux-gnueabihf-` as the cross prefix.
- Auto-add `-I<sdk>/include` and `-L<sdk>/lib`.

```bash
xmake f -p cross --sdk=/home/toolchains_sdkdir
xmake
```

`-p cross` says "generic cross platform". You can also use `-p linux` for the same toolchain so that `is_plat("linux")` returns true in `xmake.lua`.

## 2. Built-in platforms (no `--sdk` gymnastics)

These platforms have shortcuts — xmake knows how to find the tools.

### Android

```bash
xmake f -p android -a arm64-v8a --ndk=~/android-ndk-r26
xmake
```

Archs: `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`. Persist the NDK path once:

```bash
xmake g --ndk=~/android-ndk-r26     # global, remembered across projects
```

Optional: `--ndk_sdkver=21` pins the minimum API level.

### iOS / macOS / tvOS / watchOS

```bash
xmake f -p iphoneos -a arm64
xmake f -p iphonesimulator -a arm64
xmake f -p macosx -a arm64          # Apple Silicon
xmake f -p appletvos -a arm64
```

`--xcode_sdkver=` / `--target_minver=` pin the SDK and minimum deployment target.

### MinGW (Windows binaries from any host)

```bash
xmake f -p mingw --mingw=/opt/llvm-mingw -a x86_64
xmake f -p mingw -a arm64           # llvm-mingw supports arm/arm64
```

On macOS xmake usually finds Homebrew's `mingw-w64` automatically. Persist globally:

```bash
xmake g --mingw=/opt/llvm-mingw
```

### WebAssembly (emcc)

```bash
xmake f -p wasm                      # uses the built-in emcc toolchain
xmake                                # outputs .html + .wasm
```

Or target the emcc toolchain without the wasm-platform tweaks:

```bash
xmake f --toolchain=emcc
```

### Cross (the generic escape hatch)

```bash
xmake f -p cross --sdk=/opt/arm-linux-musleabi-cross -a arm
```

All other `--cc`/`--cxx`/`--cross`/`--bin` flags below also work under `-p cross`.

## 3. The full `xmake f` cross flag set

Everything below goes on `xmake f` and is persisted into `.xmake/` for subsequent builds.

| Flag | Purpose |
| --- | --- |
| `-p <plat>` | Target platform: `cross`, `linux`, `android`, `iphoneos`, `mingw`, `wasm`, custom |
| `-a <arch>` | Target arch: `x86_64`, `arm`, `arm64`, `arm64-v8a`, `riscv64`, … |
| `--sdk=` | Toolchain SDK root (has `bin/`, `include/`, `lib/`) |
| `--bin=` | Toolchain `bin` dir (when it is *not* `<sdk>/bin`) |
| `--cross=` | Cross prefix (e.g. `armv7-linux-`), when there are multiple in one `bin/` |
| `--toolchain=` | Pick a built-in/custom toolchain (`clang`, `llvm`, `gnu-rm`, `myclang`, …) |
| `--cc=`, `--cxx=` | Override C / C++ compiler binary |
| `--ld=`, `--sh=`, `--ar=` | Override executable linker / shared-lib linker / archiver |
| `--as=`, `--mm=`, `--mxx=` | Override assembler / Objective-C / Objective-C++ |
| `--strip=`, `--ranlib=` | Override strip / ranlib |
| `--includedirs=` | Extra include dirs (`:` on Unix, `;` on Windows) |
| `--linkdirs=` | Extra link dirs |
| `--links=` | Extra libraries to link (e.g. `pthread`) |
| `--cflags=` / `--cxxflags=` / `--cxflags=` / `--asflags=` | Extra compile flags |
| `--ldflags=` / `--shflags=` / `--arflags=` | Extra link / archive flags |
| `--ndk=`, `--ndk_sdkver=` | Android NDK root / API level |
| `--mingw=` | MinGW toolchain root (equivalent to `--sdk=` for `-p mingw`) |
| `--vs=`, `--vs_sdkver=`, `--vs_toolset=`, `--runtimes=` | MSVC version / Windows SDK / toolset / CRT |
| `--xcode=`, `--xcode_sdkver=`, `--target_minver=`, `--appledev=` | Xcode SDK / target version / device type |

Env-var equivalents: `CC`, `CXX`, `LD`, `AR`, `AS`, `CFLAGS`, `LDFLAGS`, etc., are honored and take precedence over the corresponding flags when set.

## 4. Cheat-sheet recipes

### Auto-detect: "just point at the SDK"

```bash
xmake f -p cross --sdk=/opt/arm-linux-gnueabihf
xmake
```

### Non-standard bin layout

`bin/` is outside the SDK root:

```bash
xmake f -p cross --sdk=/opt/sdk --bin=/opt/tools/bin
```

### Two cross prefixes in one bin dir

```
/opt/bin/
  armv7-linux-gcc
  aarch64-linux-gcc
```

Pick one with `--cross=`:

```bash
xmake f -p cross --sdk=/opt/sdk --bin=/opt/bin --cross=armv7-linux-
```

### Explicit compilers (unusual names)

```bash
xmake f -p cross --sdk=/opt/sdk \
    --cc=armv7-linux-clang --cxx=armv7-linux-clang++ \
    --ld=armv7-linux-clang++ --ar=armv7-linux-ar
```

### Compiler with a name xmake does not recognize

Use `name@path` to tell xmake "treat this binary *as* a clang++ for flag/parsing purposes":

```bash
xmake f --cxx=clang++@/home/tools/c++mips.exe
```

This is the trick for vendor compilers whose binary name xmake cannot pattern-match.

### Extra headers / libs not in `<sdk>/include` or `<sdk>/lib`

```bash
xmake f -p cross --sdk=/opt/sdk \
    --includedirs=/opt/sdk/custom/include \
    --linkdirs=/opt/sdk/custom/lib \
    --links=pthread
```

Multiple dirs: `:` on Linux/macOS, `;` on Windows.

### Ad-hoc flags

```bash
xmake f -p cross --sdk=/opt/sdk \
    --cflags="-DTEST -I/xxx/xxx" \
    --ldflags="-lpthread"
```

### Switch toolchain on a host platform (no cross)

```bash
xmake f --toolchain=clang
xmake f --toolchain=gcc-11
xmake f --toolchain=llvm --sdk=/opt/llvm-18
xmake f --toolchain=gnu-rm --sdk=/opt/gcc-arm-none-eabi
xmake f --toolchain=tinyc
xmake f --toolchain=icc
xmake f --toolchain=ifort
```

See the full list:

```bash
xmake show -l toolchains
```

## 5. Custom toolchains in `xmake.lua`

When command-line flags are not enough (or you want to check a toolchain into the repo), define a toolchain in `xmake.lua`.

### Minimal: just an SDK path

```lua
toolchain("my_toolchain")
    set_kind("standalone")
    set_sdkdir("/opt/arm-linux-musleabi-cross")
toolchain_end()
```

Use it:

```bash
xmake f --toolchain=my_toolchain
xmake
```

`set_kind("standalone")` means "this replaces the whole toolchain" — xmake will not mix it with the host compiler.

### Per-target toolchain (no CLI switch needed)

```lua
toolchain("my_toolchain")
    set_kind("standalone")
    set_sdkdir("/opt/arm-linux-musleabi-cross")
toolchain_end()

target("firmware")
    set_kind("binary")
    set_plat("cross")
    set_arch("arm")
    set_toolchains("my_toolchain")
    add_files("src/*.c")
```

Now `xmake` automatically builds `firmware` with `my_toolchain`, while other targets can still use the host compiler. Great for mixed-build repos (host tool + firmware in one project).

### Full toolset override

```lua
toolchain("myclang")
    set_kind("standalone")
    set_toolset("cc",    "clang")
    set_toolset("cxx",   "clang", "clang++")
    set_toolset("ld",    "clang++", "clang")
    set_toolset("sh",    "clang++", "clang")
    set_toolset("ar",    "ar")
    set_toolset("strip", "strip")
    set_toolset("as",    "clang")
    set_toolset("mm",    "clang")
    set_toolset("mxx",   "clang", "clang++")

    add_defines("MYCLANG")

    on_check(function (toolchain)
        import("lib.detect.find_tool")
        return find_tool("clang")
    end)

    on_load(function (toolchain)
        local march = is_arch("x86_64", "x64") and "-m64" or "-m32"
        toolchain:add("cxflags", march)
        toolchain:add("ldflags", march)
        toolchain:add("shflags", march)
        toolchain:add("asflags", march)
    end)
toolchain_end()
```

- `set_toolset("cc", ...)` accepts multiple candidates — xmake picks the first that exists.
- `on_check` gates whether the toolchain is usable on this host.
- `on_load` injects flags once the toolchain is selected (arch flags, extra include/link dirs, defines).

Available `set_toolset` keys: `cc`, `cxx`, `ld`, `sh`, `ar`, `ex`, `strip`, `ranlib`, `as`, `mm`, `mxx`, `sc` (Swift), `dc` (D), `gc` (Go), `rc` (Rust), `fc` (Fortran), `cu` (CUDA), `cu-ld`, `cu-ccbin`, `cpp` (preprocessor), `dlltool`, `mrc`.

## 6. Debugging cross-compilation setup

- Show the detected config:

  ```bash
  xmake f -p cross --sdk=/opt/sdk -vD
  xmake -v                  # see actual compile commands
  ```

  Look for `cc`, `cxx`, `ld`, `cross`, `bin`, `includedirs`, `linkdirs` in the config dump.

- List toolchains currently visible to xmake:

  ```bash
  xmake show -l toolchains
  xmake show -l plats
  ```

- Inspect what xmake actually runs for one target:

  ```bash
  xmake build -v <target>
  ```

- If a tool-detection probe hangs, set `XMAKE_PROFILE=stuck xmake f -c` to see which subprocess it's stuck on (see `xmake-troubleshooting`).

## 7. Common pitfalls

- **`--sdk=` points too deep.** Give xmake the root that has `bin/include/lib`, not `bin/` itself.
- **Multiple cross prefixes in one `bin/`.** Auto-detection picks the first one it finds; use `--cross=` to disambiguate.
- **Unknown compiler binary name.** If the vendor ships `c++mips.exe` instead of `gcc`/`clang`/`g++`, use `--cxx=clang++@/path/to/c++mips.exe` so xmake knows which flag dialect to use.
- **Env vars overriding your flags.** If `CC=gcc` is exported, it beats `--cc=arm-linux-gcc`. `unset CC CXX LD AR AS` before configuring.
- **Forgetting to re-configure after changing SDKs.** `xmake f -c` after any SDK path change; stale cache will still use the old tools.
- **`-p linux` vs `-p cross`.** Functionally identical for cross-compiling, but `is_plat("linux")` branches in `xmake.lua` only trigger on `-p linux`.
- **Packages built for the wrong arch.** If you cross-compile with `add_requires(...)`, xmake rebuilds those packages for the target by default — but stale ones in `~/.xmake/packages` can persist. `xmake require --force <pkg>` to rebuild.

## When to branch out

- Host-only compiler switching (gcc ↔ clang on your laptop) → `xmake-toolchains`
- Building packages cross-compiled for the target → `xmake-packages`, `xmake-repo-testing`
- Hang/stuck during tool detection → `xmake-troubleshooting`
