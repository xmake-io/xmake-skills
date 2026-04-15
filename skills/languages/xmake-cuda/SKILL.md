---
name: xmake-cuda
description: Use when building CUDA projects with xmake — `.cu` sources, `--cuda=` / `--cuda_sdkver=` SDK pinning, GPU architecture selection, device linking (`build.cuda.devlink`), and mixing CUDA with C++ targets.
---

# Building CUDA with Xmake

Xmake detects CUDA automatically and handles device code + host code compilation, dependency scanning, and device linking.

## 1. Minimal project

```bash
xmake create -P test -l cuda
cd test
xmake
```

```lua
add_rules("mode.debug", "mode.release")

target("test")
    set_kind("binary")
    add_files("src/*.cu")
```

## 2. Target kinds

```lua
target("app")         set_kind("binary")
target("staticcuda")  set_kind("static")
target("sharedcuda")  set_kind("shared")
```

## 3. SDK / version selection

```bash
xmake f --cuda=/usr/local/cuda-12.2                 -- specific install
xmake f --cuda=12.2                                 -- version; looks up default install
xmake f --cuda_sdkver=11.8                          -- v3.0.5+, per-project SDK version
xmake f --cuda_sdkver=11.x                          -- any 11.x
xmake f --cuda_sdkver=auto                          -- auto-detect (default)
```

Persist globally:

```bash
xmake g --cuda=/usr/local/cuda-12.2
```

## 4. GPU architecture

```lua
target("app")
    add_files("src/*.cu")
    add_cugencodes("sm_70", "sm_86", "sm_90")          -- compile for these archs
    add_cugencodes("native")                           -- just the host GPU
```

`add_cugencodes` takes `sm_XX` strings or `native`. Multiple calls accumulate; xmake emits `-gencode=arch=compute_XX,code=sm_XX` per entry.

## 5. Device linking

Device-linking is automatic for `binary` and `shared` targets:

```lua
target("app")
    set_kind("binary")
    add_files("src/*.cu")
    -- device link runs automatically
```

Disable it (rarely needed):

```lua
set_policy("build.cuda.devlink", false)
```

### `static` targets — manual device link

Static libraries are **not** device-linked by default. If a downstream binary has no `.cu` files but depends on a static cuda lib, you'll hit "undefined reference to __device_..." errors. Fix by opting in:

```lua
target("cudalib")
    set_kind("static")
    add_files("src/*.cu")
    add_values("cuda.build.devlink", true)         -- force device link for this static target
```

## 6. Mixing CUDA with C++

```lua
target("app")
    set_kind("binary")
    set_languages("c++17")
    add_files("src/*.cpp", "src/*.cu")
    add_cugencodes("sm_80")
```

Host code in `.cpp` and device code in `.cu` mix freely in one target. Xmake invokes `nvcc` for `.cu` and the normal C++ compiler for `.cpp`.

## 7. Flags

```lua
target("app")
    add_files("src/*.cu")
    add_cuflags("--use_fast_math", "-lineinfo")       -- nvcc flags
    add_cuflags("-Xcompiler=-fPIC", {force = true})   -- flags passed to host compiler via nvcc
```

`add_cuflags` = nvcc (cuda compiler) flags. For flags that must reach the host compiler (gcc/clang/msvc), use `-Xcompiler=...`.

## 8. Cross-compile / Jetson / Tegra

```bash
xmake f -p linux -a arm64 --cuda=/usr/local/cuda-cross-aarch64
xmake
```

Point `--cuda` at a cross CUDA SDK. Jetson deployment targets generally set `-a arm64`.

## Pitfalls

- **`sm_xx` vs compute capability.** `sm_86` means "Ampere RTX 30xx". `add_cugencodes` takes the binary code name, not the compute cap number.
- **Static lib with no device link.** Undefined device-symbol errors at final link. `add_values("cuda.build.devlink", true)` on the static target.
- **Mixing hosts.** Every `.cu` file gets compiled through `nvcc`, which calls the host compiler. If you mix toolchains (clang + nvcc that expects gcc), you hit header incompatibilities. Stick with the nvcc-supported host compiler for the CUDA version.
- **CUDA version vs host compiler version.** Each CUDA release supports a specific range of gcc/clang/MSVC. Check NVIDIA's docs — "gcc 13 unsupported by CUDA 11.8" is a common surprise.
- **Too many `sm_` targets.** Each `add_cugencodes` entry doubles compile time. List only the GPUs you actually target.

## When to branch out

- Cross-compile generic plumbing → `xmake-cross-compilation`
- Target basics, host C++ compilation → `xmake-targets`
- Policies (including `build.cuda.devlink`) → `xmake-policy`
