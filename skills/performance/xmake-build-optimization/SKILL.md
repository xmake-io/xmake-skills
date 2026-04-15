---
name: xmake-build-optimization
description: Use when a project builds slowly and you want to make it faster — parallel jobs, unity build, precompiled headers, build cache, LTO, link-time improvements, binary package cache, and profiling with `XMAKE_PROFILE=perf:*`. Covers idiomatic xmake speedups and the order to try them.
---

# Xmake Build Optimization & Best Practices

A roadmap for making an xmake build faster. Try them in order — each has a different cost/benefit. Measure before and after with `XMAKE_PROFILE=perf:tag` (or `time xmake`).

## 0. Measure first

Don't guess where the time goes:

```bash
XMAKE_PROFILE=perf:tag     xmake        # compile vs link vs other
XMAKE_PROFILE=perf:process xmake        # which subprocesses dominate
XMAKE_PROFILE=perf:call    xmake        # expensive Lua functions
XMAKE_PROFILE=trace        xmake f -c   # trace every subprocess in configure
time xmake --rebuild                    # total wall-clock
```

Profile before each change — "speedups" often turn out to be in the cache-hit, not the fix.

See `xmake-troubleshooting` for the full profiler reference.

## 1. Parallelism

```bash
xmake -j N                              # N parallel jobs
xmake -j $(nproc)
xmake -j $(sysctl -n hw.ncpu)           # macOS
```

Default is already `⌈3/2 × ncpu⌉`. Override for:

- Huge machines where xmake's default is too conservative.
- Memory-limited builds where you need to reduce parallelism (big LTO links can OOM).

## 2. Local compile cache (on by default)

```bash
xmake f --ccache=y          # default
xmake f --ccache=n          # disable for debugging
```

Xmake ships a built-in cache since 2.6.6 — faster than external ccache, cross-platform (MSVC too). See `xmake-build-cache`.

Verify it's working — look for `cache compiling.release` in the output.

## 3. Unity builds

Compile multiple source files as one translation unit. Massive speedup (2–5×) for projects with hundreds of small files.

```lua
target("mylib")
    add_rules("c++.unity_build", {batchsize = 8})    -- 8 files per unit
    add_rules("c.unity_build")
    add_files("src/*.cpp")
```

Pitfalls (usual unity-build ones):

- File-local `static` symbols clash with same-named symbols in another file in the same unit. Rename or exclude.
- Anonymous namespaces collide similarly.
- Order-dependent macros leak across files in the same unit.

Exclude problem files:

```lua
add_files("src/*.cpp")
add_files("src/legacy.cpp", {unity_ignored_files = true})
```

## 4. Precompiled headers (PCH)

```lua
target("app")
    set_pcxxheader("src/pch.hpp")       -- C++ PCH
    -- or:
    set_pcheader("src/pch.h")           -- C PCH
    add_files("src/*.cpp")
```

Good fit when many files include the same heavy STL/boost/Qt header. PCH is compiled once and reused.

## 5. Reduce the work

Before optimizing the compiler, remove files you don't need:

- **Scope includes correctly.** `add_includedirs("include", {public = true})` only for the actual public dir — not `src/`.
- **Split targets.** Libraries that rarely change stay cached; split noisy code into its own target so it doesn't invalidate the rest.
- **Avoid per-file defines.** `add_defines` at target level is parsed once; `target:add` inside `on_load` runs per-target but is still cheaper than per-file.
- **Glob, don't enumerate.** `add_files("src/**/*.cpp")` lets xmake hash once; listing 500 files explicitly is slower to parse.
- **`set_policy("build.warning", false)` when CI checks warnings** — don't re-emit warning diagnostics in the inner loop.

## 6. Binary package cache

If you use lots of packages, pull precompiled binaries instead of building from source:

```bash
xrepo install --precompiled fmt spdlog boost
```

Or project-level: packages that have prebuilt binaries are used automatically when available. `xmake require --info <pkg>` shows whether the precompiled variant exists.

Cache package installs in CI:

```yaml
# GitHub Actions
- uses: actions/cache@v4
  with:
    path: |
      ~/.xmake/packages
      ~/.xmake/cache
    key: xmake-${{ runner.os }}-${{ hashFiles('xmake.lua') }}
```

## 7. Link-time improvements

Linking often dominates build time for big projects.

### LTO (release only)

```lua
target("app")
    if is_mode("release") then
        set_policy("build.optimization.lto", true)
    end
```

LTO makes release builds slower but smaller/faster at runtime. Don't enable for debug builds.

### Use `lld` instead of `ld`

```bash
xmake f --ldflags="-fuse-ld=lld"        # Linux/macOS with clang
xmake f --ldflags="-fuse-ld=mold"       # even faster (Linux)
```

`mold` and `lld` can be 5–10× faster than BFD ld on large links.

### Split shared libraries

Huge monolithic static libs link slowly. Split into a few `shared` or `static` targets with proper `add_deps` — xmake will only re-link what changed.

### Symbol visibility

Hide symbols by default to shrink link work:

```lua
set_symbols("hidden")
-- or flags:
add_cxflags("-fvisibility=hidden", "-fvisibility-inlines-hidden")
```

## 8. Debug-build-specific speedups

Release isn't the bottleneck for daily dev — debug is.

```lua
if is_mode("debug") then
    set_optimize("none")                    -- or "fast" for balanced
    set_symbols("debug")
    add_cxflags("-g1", {force = true})      -- minimal debug info
end
```

- `-g1` instead of `-g` cuts debug info size drastically.
- Split DWARF: `add_cxflags("-gsplit-dwarf")` + `add_ldflags("-Wl,--gdb-index")`.
- `-fno-inline` off if you don't need to step into inlined functions.

## 9. Distributed and remote caches

Scale beyond one machine:

- **Distributed compile** (`--distcc`) — farm out compile jobs to a cluster. See `xmake-distributed-compilation`.
- **Remote cache** (`--ccache`) — share compiled objects across a team/CI. See `xmake-build-cache`.

Both compose with the local cache and each other. Typical team setup:

```bash
xmake service --connect --distcc --ccache
```

## 10. Keep xmake's own overhead low

The description-domain parse time and tool detection can dominate tiny projects.

- **Don't put I/O in description domain.** See `xmake-style`. Every parse re-runs top-level code.
- **Avoid `os.iorun` / `git` at top level.** Use `on_load` or a task.
- **`xmake f` once, not per file.** Configure caches to `.xmake/`; don't re-configure inside tight loops.
- **Avoid deep `add_moduledirs` chains** during steady-state builds — only needed while iterating on native modules.

## Order to try (cheat sheet)

1. `XMAKE_PROFILE=perf:tag xmake` — **know where time goes first**.
2. `-jN` — parallelism.
3. Verify `--ccache=y` and look for cache hits.
4. Unity build on the biggest target.
5. PCH for common headers.
6. Binary packages (`--precompiled`, CI cache for `~/.xmake/packages`).
7. `lld` / `mold` linker.
8. Debug-build-specific flag trimming.
9. LTO (release only).
10. Distcc + remote cache (team/CI scale).

## Common anti-patterns

- **Optimizing release when dev is debug.** Profile the build you actually run.
- **Enabling LTO in debug builds.** Makes debug slower for no gain.
- **Disabling the cache to "get clean times".** Fine for benchmarks, not for dev loops.
- **Huge monolithic targets.** Changes anywhere → rebuild everything → re-link everything.
- **Unity build + anonymous namespaces collision.** Check unity-friendliness before enabling.
- **Turning everything on blindly.** LTO + PCH + unity + `-g3` can fight each other. Add one knob at a time.

## When to branch out

- Cache details (local / remote) → `xmake-build-cache`
- Multi-machine job pooling → `xmake-distributed-compilation`
- Compiling on a remote box → `xmake-remote-compilation`
- Slowness is actually a hang → `xmake-troubleshooting`
- `xmake.lua` parsing is slow → `xmake-style`
