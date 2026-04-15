---
name: xmake-trybuild
description: Use when you want to build a third-party project that uses CMake, Autotools, Meson, Ninja, Bazel, Make, MSBuild, Xcodebuild, or NdkBuild — through xmake, so you inherit xmake's cross-compilation and toolchain detection. Covers `xmake` auto-detection, `--trybuild=<system>`, `--tryconfigs="..."`, and cross-compile recipes for Android / iOS / MinGW / custom SDKs.
---

# Xmake TryBuild — Drive Third-Party Build Systems

`trybuild` lets xmake invoke **another build system** (CMake, Autotools, Meson, …) while still providing its own platform/toolchain detection, cross-compile support, and incremental-build wrappers. Use it when you need to build an upstream project that doesn't use `xmake.lua`, but still want xmake's cross-compilation ergonomics.

## Supported backends

| `--trybuild=` | Backend |
| --- | --- |
| `autotools` | `./configure` + `make` (cross-compile supported) |
| `cmake` | `cmake` + generator (cross-compile supported) |
| `meson` | `meson` + `ninja` |
| `ninja` | direct `ninja` |
| `make` | `make` / `Makefile` |
| `msbuild` | Visual Studio MSBuild |
| `xcodebuild` | Xcode CLI builds |
| `scons` | SCons |
| `bazel` | Bazel |
| `ndkbuild` | Android NDK ndk-build |

## 1. Auto-detect mode

Run plain `xmake` inside a project root that has a recognized file (`CMakeLists.txt`, `configure`, `Makefile`, `meson.build`, …). Xmake prompts before building:

```bash
$ cd libpng-1.6.35
$ xmake
note: CMakeLists.txt found, try building it (pass -y or --confirm=y/n/d to skip confirm)?
please input: y (y/n)
-- Configuring done
-- Generating done
...
output to /Users/ruki/Downloads/libpng-1.6.35/build/artifacts
build ok!
```

Add `-y` to skip the prompt in CI:

```bash
xmake -y
```

Artifacts land under `build/artifacts/`.

## 2. Force a specific backend

When a project ships multiple build systems (e.g. both `configure` and `CMakeLists.txt`):

```bash
xmake f --trybuild=cmake
xmake
```

Once set, subsequent `xmake` / `xmake clean` calls use the configured backend — no re-prompting.

## 3. Standard xmake commands work

Once trybuild is configured, normal xmake commands drive the upstream build:

```bash
xmake                    # build
xmake --rebuild          # force full rebuild
xmake clean              # clean objects
xmake clean --all        # wipe everything (configure cache, artifacts)
xmake f -m debug         # switch to debug build
xmake f -m release
```

Behavior is mapped onto each backend — e.g. for autotools, `xmake clean --all` runs `make distclean`.

## 4. Cross-compilation via trybuild

The killer feature: xmake's cross-compilation config works for CMake/Autotools projects too.

### Android

```bash
xmake f -p android --trybuild=cmake [--ndk=/path/to/ndk]
xmake
```

Equivalent of setting up `CMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake` + `ANDROID_ABI` + `ANDROID_NATIVE_API_LEVEL` by hand.

```bash
xmake f -p android --trybuild=autotools [--ndk=/path/to/ndk]
xmake
```

Equivalent of exporting `CC`, `CXX`, `AR`, `LD`, `RANLIB`, `STRIP` and passing `--host=aarch64-linux-android` to `./configure`.

### iOS / macOS

```bash
xmake f -p iphoneos --trybuild=cmake
xmake

xmake f -p iphoneos --trybuild=autotools
xmake
```

### MinGW (Windows from any host)

```bash
xmake f -p mingw --trybuild=cmake [--mingw=/opt/llvm-mingw]
xmake
```

### Generic cross-SDK

```bash
xmake f -p cross --trybuild=cmake --sdk=/opt/arm-linux-gnueabihf
xmake
```

All the cross flags from `xmake-cross-compilation` work identically with `--trybuild=...` appended.

## 5. Passing arguments to the backend

### `--tryconfigs`

Extra flags forwarded to the backend's config step:

```bash
# autotools: passed to ./configure
xmake f --trybuild=autotools --tryconfigs="--enable-shared=no --disable-tests"

# cmake: passed to cmake
xmake f --trybuild=cmake --tryconfigs="-DBUILD_SHARED_LIBS=ON -DBUILD_TESTING=OFF"

# meson: passed to meson setup
xmake f --trybuild=meson --tryconfigs="-Doption=value"
```

### Compile/link flag passthrough

Built-in xmake flags automatically propagate — no `--tryconfigs` needed for these:

```bash
xmake f --trybuild=cmake --cflags="-DNDEBUG" --ldflags="-lm" --includedirs=/opt/xyz/include
xmake
```

## 6. Android JNI (`ndkbuild`)

If `jni/Android.mk` exists:

```bash
xmake f -p android --trybuild=ndkbuild [--ndk=...]
xmake
```

For gradle integration see the [xmake-gradle plugin](https://github.com/xmake-io/xmake-gradle).

## 7. Typical recipes

### Build a CMake project for the host

```bash
cd upstream-project
xmake f --trybuild=cmake --tryconfigs="-DBUILD_SHARED_LIBS=ON"
xmake
ls build/artifacts
```

### Cross-compile libpng for Android

```bash
cd libpng-1.6.40
xmake f -p android -a arm64-v8a --trybuild=cmake --ndk=~/android-ndk-r26 \
        --tryconfigs="-DPNG_SHARED=ON -DPNG_STATIC=OFF"
xmake
```

### Autotools project for MinGW from Linux

```bash
cd autotools-thing
xmake f -p mingw --mingw=/usr/x86_64-w64-mingw32 --trybuild=autotools \
        --tryconfigs="--disable-tests"
xmake
```

### Meson + Ninja for the host

```bash
xmake f --trybuild=meson
xmake -j $(nproc)
```

## 8. When to prefer trybuild vs a package recipe

- **One-off local build of upstream code** → trybuild.
- **Reusable dependency consumed by `add_requires`** → write a package recipe that uses `import("package.tools.cmake")` (or `autoconf`, `meson`) — see `xmake-private-packages` and `xmake-repo-testing`. The recipe gets proper version pinning, hash checks, and caching, which trybuild does not.

## Pitfalls

- **Missing backend tool.** Trybuild shells out to `cmake`/`autotools`/`meson`/… — make sure the binary is installed. On failure, `xmake -vD` shows the missing tool.
- **Auto-detect picks the wrong backend.** Projects that ship both `Makefile` and `CMakeLists.txt` may auto-select the "wrong" one; use `xmake f --trybuild=cmake` to pin.
- **Clean doesn't clean configure cache.** `xmake clean` removes objects only. Use `xmake clean --all` to also wipe the backend's configure state.
- **Cross-compile of Bazel / SCons.** Not supported to the same depth as CMake/Autotools — check `--trybuild=`'s doc for the given backend.
- **`--tryconfigs` doesn't support `=` inside quoted values easily.** Cross-shell quoting can be tricky; prefer multiple small flags.
- **Artifacts path is backend-specific.** Usually `build/artifacts/`, but some backends emit to their own layout. Look for the "output to ..." line in `xmake` output.

## When to branch out

- Cross toolchain flags (`--sdk`/`--ndk`/`--mingw`/`--cross`) → `xmake-cross-compilation`
- Packaging *upstream* code as a reusable dependency → `xmake-private-packages` / `xmake-repo-testing`
- Full xmake CLI reference → `xmake-commands`
