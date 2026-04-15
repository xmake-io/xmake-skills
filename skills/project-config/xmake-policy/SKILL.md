---
name: xmake-policy
description: Use when setting build policies via `set_policy("name", value)` — named feature toggles that control sanitizers, LTO, warnings, C++ modules, ccache, module-reuse, rpath, preprocessor behavior, package download strict mode, and many others. Covers the common policies grouped by purpose, how to scope them (project vs target), and where to find the full list.
---

# set_policy — Feature Toggles

Policies are named feature toggles xmake exposes via `set_policy("<name>", <value>)`. They replace ad-hoc flags with a stable, versioned API — enabling a feature like "C++20 modules" or "address sanitizer" is a single call that works across compilers.

Policies live in `xmake.lua` and can be set at project or target scope.

## 1. Basic usage

```lua
-- project-wide
set_policy("build.warning", true)

target("app")
    -- per-target
    set_policy("build.optimization.lto", true)
    set_policy("build.sanitizer.address", true)
```

- Project-scope (at top of `xmake.lua`) — applies to every target.
- Target-scope (inside `target(...)`) — only that target.
- Target values override the project value.

Reading a policy from script:

```lua
on_load(function (target)
    if target:policy("build.warning") then
        ...
    end
end)
```

## 2. Commonly-used policies, grouped

### Compilation / warnings

```lua
set_policy("build.warning",             true)        -- show compile warnings (default off)
set_policy("check.auto_ignore_flags",   false)       -- don't auto-filter unknown flags
set_policy("check.auto_map_flags",      false)       -- don't auto-map gcc→msvc flag names
```

### Optimization / linking

```lua
set_policy("build.optimization.lto",    true)        -- enable LTO
set_policy("build.merge_archive",       true)        -- merge all static archives into one
set_policy("build.release.strip",       true)        -- strip release binaries (v3.0.8+)
set_policy("build.rpath",               true)        -- auto-emit rpath for shared deps
set_policy("install.rpath",             true)        -- same at install time
set_policy("build.intermediate_directory", true)     -- use an intermediate dir per target
```

### Sanitizers (GCC/Clang)

```lua
set_policy("build.sanitizer.address",   true)        -- -fsanitize=address
set_policy("build.sanitizer.thread",    true)        -- -fsanitize=thread
set_policy("build.sanitizer.memory",    true)        -- -fsanitize=memory
set_policy("build.sanitizer.leak",      true)        -- -fsanitize=leak
set_policy("build.sanitizer.undefined", true)        -- -fsanitize=undefined
```

Mutually exclusive — enable one at a time. Usually gated by mode:

```lua
if is_mode("asan") then
    set_policy("build.sanitizer.address", true)
end
```

### C++20 modules

```lua
set_policy("build.c++.modules",                 true)     -- enable module build (required)
set_policy("build.c++.modules.std",             true)     -- enable `import std;`
set_policy("build.c++.modules.culling",         true)     -- drop unused modules
set_policy("build.c++.modules.reuse",           true)     -- reuse BMIs across targets
set_policy("build.c++.modules.reuse.strict",    true)
set_policy("build.c++.modules.fallbackscanner", true)     -- use fallback dep scanner
```

See `xmake-cxx-modules`.

### Cache / ccache

```lua
set_policy("build.ccache",                true)     -- enable built-in cache
set_policy("build.ccache.global_storage", true)     -- share cache across projects
```

### Parallelism

```lua
set_policy("build.across_targets_in_parallel", true)   -- build sibling targets in parallel
set_policy("build.fence",                      true)   -- use fence barriers between phases
set_policy("build.distcc.remote_only",         true)   -- force all jobs to distcc server
```

### Packages

```lua
set_policy("package.requires_lock",                 true)    -- lockfile-based pinning
set_policy("package.precompiled",                   true)    -- prefer binary packages
set_policy("package.fetch_only",                    true)    -- fetch, don't install
set_policy("package.install_only",                  true)    -- install, don't link
set_policy("package.install_always",                true)    -- reinstall each configure
set_policy("package.strict_compatibility",          true)    -- abi-strict pkg matching
set_policy("package.librarydeps.strict_compatibility", true)
set_policy("package.download.http_headers", {"Authorization: Bearer xyz"})
```

### Preprocessor

```lua
set_policy("preprocessor.linemarkers",           true)
set_policy("preprocessor.gcc.directives_only",   true)      -- for ccache compatibility
```

### Windows-specific

```lua
set_policy("windows.manifest.uac",     "invoker")           -- asInvoker | highestAvailable | requireAdministrator
set_policy("windows.manifest.uac.ui",  true)
```

### Runtime / CUDA

```lua
set_policy("run.windows_error_dialog", false)               -- disable popup on crash
set_policy("run.autobuild",            true)                -- auto-rebuild before xmake run
set_policy("build.cuda.devlink",       true)                -- CUDA device-link step
```

### Debug

```lua
set_policy("build.c++.dynamic_debugging", true)             -- enable dynamic debugging (v3.0.6+)
```

## 3. Finding every policy

The full list lives in xmake's source:

```bash
xmake l core.base.policy.policies
```

Or browse `xmake-docs`: **API → description → Built-in Policies** has the canonical list with version badges (`v2.3.4+`, etc.). Use the badge to check minimum xmake version for a given policy.

## 4. Setting via command line

Most policies can also be set transiently on `xmake f`:

```bash
xmake f --policies="build.sanitizer.address"
xmake f --policies="build.optimization.lto,build.warning"
```

Comma-separated list; booleans default to `true`. Useful for one-off CI runs without touching `xmake.lua`.

## 5. Conditional application

```lua
target("app")
    if is_mode("release") then
        set_policy("build.optimization.lto", true)
        set_policy("build.release.strip", true)
    elseif is_mode("debug") then
        set_policy("build.warning", true)
    elseif is_mode("asan") then
        set_policy("build.sanitizer.address", true)
    end
```

Don't enable LTO in debug — makes builds slower with no benefit.

## 6. Reading policy state in scripts

```lua
on_config(function (target)
    if target:policy("build.c++.modules") then
        -- do module-specific config
    end
end)
```

`target:policy("name")` returns the effective value (target policy overriding project policy).

## 7. Defining a custom policy

You can register your own policies for use by custom rules:

```lua
-- at top of xmake.lua
import("core.base.policy")
policy.register("mycompany.enable_hardening", "boolean")

target("app")
    set_policy("mycompany.enable_hardening", true)

rule("hardening")
    on_config(function (target)
        if target:policy("mycompany.enable_hardening") then
            target:add("cxflags", "-fstack-protector-strong", "-D_FORTIFY_SOURCE=2")
        end
    end)
```

Types: `"boolean"`, `"string"`, `"number"`, `"table"`.

## Pitfalls

- **Policy vs flag.** `set_policy("build.optimization.lto", true)` is portable; `add_cxflags("-flto")` is not — use the policy.
- **Version checks.** Each policy has a "since" version. Using a newer policy on an older xmake silently succeeds (or raises) — check the version badge when reading docs, and pin `set_xmakever("2.8.0")` at the top of your `xmake.lua` if needed.
- **Sanitizers together.** Most sanitizers are mutually exclusive (ASan + TSan can't run together). Enable one per build mode.
- **Project policy hidden by target.** A target's `set_policy` replaces the project value entirely. If you want to merge, handle it in `on_load`.
- **`package.install_always`.** Reinstalls every configure — don't leave it on in shared config; kills iteration speed.
- **`build.warning` default is off.** Many users are surprised by this. Turn it on at the project level during dev, off in CI for stable outputs.

## When to branch out

- Target compile flags and visibility → `xmake-targets`
- C++ modules setup → `xmake-cxx-modules`
- Sanitizers in practice → `xmake-build-optimization`
- Package behavior toggles → `xmake-packages`, `xmake-private-packages`
- Custom rules reading policies → `xmake-rules`, `xmake-scripting`
