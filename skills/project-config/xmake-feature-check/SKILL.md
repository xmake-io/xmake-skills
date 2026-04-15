---
name: xmake-feature-check
description: Use when probing the host/toolchain for capabilities inside `xmake.lua` — `has_cfuncs` / `has_cxxtypes` / `has_features`, `check_csnippets`, `check_cflags` / `check_cxxflags`, option auto-probes (`add_cxxincludes` / `add_cfuncs`), and `lib.detect.find_*`. Covers how to gate target config on whether a symbol/header/flag is actually usable on the target platform.
---

# Feature & Capability Checks

Xmake can probe the target toolchain at configure time for:

- Does a **compiler flag** work? (`check_cflags`, `check_cxxflags`)
- Does a **C/C++ function** exist? (`has_cfuncs`, `has_cxxfuncs`)
- Does a **type** exist? (`has_ctypes`, `has_cxxtypes`)
- Does an **include** exist? (`has_cincludes`, `has_cxxincludes`)
- Does an arbitrary **snippet** compile? (`check_csnippets`, `check_cxxsnippets`)
- Does a C++ **feature** work? (`has_features("cxx_constexpr")`, …)
- Is a **tool/program** on `PATH`? (`find_tool`, `find_program`)
- Is a **library** findable? (`find_package`)

Use these to branch configuration on capability instead of hard-coding `if is_plat(...)`.

## In an option (the declarative form)

Options have a short-hand for capability probes — if the probe succeeds, the option is automatically enabled.

```lua
option("has_avx2")
    add_cxflags("-mavx2")
    add_cxxincludes("immintrin.h")
    add_cxxsnippets("int f(){ return _mm256_set1_epi32(1)[0]; }")
option_end()

target("app")
    if has_config("has_avx2") then
        add_cxflags("-mavx2")
        add_defines("HAVE_AVX2=1")
    end
```

At configure time xmake compiles the snippet with `-mavx2` and the header — the option becomes `true` if the whole probe succeeds, `false` otherwise. Transparently handles cross-compile, different compilers, and cached results.

Short-hands available in `option()`:

| API | Probes |
| --- | --- |
| `add_cincludes("x.h")` | `#include <x.h>` compiles |
| `add_cxxincludes("x.hpp")` | C++ header is usable |
| `add_ctypes("ssize_t")` | type exists |
| `add_cxxtypes("std::filesystem::path")` | C++ type exists |
| `add_cfuncs("pthread_create")` | function is linkable |
| `add_cxxfuncs("std::thread")` | C++ function/class is usable |
| `add_csnippets("snippet code")` | arbitrary C snippet compiles |
| `add_cxxsnippets("snippet code")` | arbitrary C++ snippet compiles |
| `add_cflags` / `add_cxflags` | flag probe |
| `add_features("c_static_assert")` | named compiler feature |
| `add_links("foo")` | library links |

The option succeeds only if **all** configured probes pass.

## In a script hook (the imperative form)

For complex logic, probe from `on_config` / `on_load` using `lib.detect.*`:

```lua
target("app")
    on_config(function (target)
        import("lib.detect.check_cxsnippets")
        import("lib.detect.has_cfuncs")

        if has_cfuncs("pthread_create", {includes = "pthread.h"}) then
            target:add("syslinks", "pthread")
            target:add("defines", "HAVE_PTHREAD")
        end

        if check_cxsnippets([[
            #include <filesystem>
            int main() { std::filesystem::path p; return 0; }
        ]], {configs = {languages = "c++17"}}) then
            target:add("defines", "HAVE_STD_FILESYSTEM")
        end
    end)
```

Common detect modules:

```lua
import("lib.detect.find_tool")          -- program on PATH / SDK
import("lib.detect.find_program")       -- lower-level program probe
import("lib.detect.find_package")       -- library pkg-config / xmake lookup
import("lib.detect.find_library")       -- find a library file
import("lib.detect.find_file")          -- find a file in search dirs
import("lib.detect.find_path")          -- find a dir that contains a file
import("lib.detect.find_cudadevices")   -- CUDA devices
import("lib.detect.has_flags")          -- does the compiler accept this flag?
import("lib.detect.has_cfuncs")
import("lib.detect.has_cxxfuncs")
import("lib.detect.has_ctypes")
import("lib.detect.has_cxxtypes")
import("lib.detect.has_cincludes")
import("lib.detect.has_cxxincludes")
import("lib.detect.has_features")
import("lib.detect.check_csnippets")
import("lib.detect.check_cxxsnippets")
import("lib.detect.check_cflags")
import("lib.detect.check_cxxflags")
import("lib.detect.check_ldflags")
```

### `find_tool`

```lua
import("lib.detect.find_tool")
local protoc = find_tool("protoc")
if protoc then
    print("found protoc at %s (version %s)", protoc.program, protoc.version)
end
```

Returns a table `{program = "...", version = "..."}` on success, `nil` on failure.

### `has_flags` / `check_cflags`

```lua
import("lib.detect.has_flags")
if has_flags("-Wshadow=local", "cxxflags") then
    target:add("cxxflags", "-Wshadow=local")
end
```

`has_flags` returns boolean; `check_cflags` returns the flag itself if supported, else `nil`. Both cache their result across runs in `.xmake/`.

### `check_cxxsnippets`

```lua
import("lib.detect.check_cxxsnippets")
local ok = check_cxxsnippets([[
    #include <coroutine>
    struct task { struct promise_type { task get_return_object(){return {};} }; };
]], {configs = {languages = "c++20"}, includes = "coroutine"})

if ok then
    target:add("defines", "HAVE_COROUTINES")
end
```

`configs` gets passed to the probe compile — `languages`, `cxflags`, `defines`, etc.

## Generating a config header from checks

```lua
option("enable_foo")
    set_default(false)

target("app")
    add_files("src/*.c")
    set_configdir("$(buildir)/config")
    add_configfiles("src/config.h.in")
    set_configvar("HAVE_STD_FILESYSTEM",
        check_cxxsnippets([[
            #include <filesystem>
            int main(){ return 0; }
        ]]) and 1 or 0)
    set_configvar("ENABLE_FOO", has_config("enable_foo") and 1 or 0)
```

`src/config.h.in`:

```c
#define HAVE_STD_FILESYSTEM ${HAVE_STD_FILESYSTEM}
#define ENABLE_FOO          ${ENABLE_FOO}
```

## Caching

All probes cache their result in `.xmake/` keyed by compiler+flag+arch. Results persist across builds. To force re-probing:

```bash
xmake f -c          # drop cached config
xmake f --check     # re-run all detections
```

## Pitfalls

- **Probe cached from the wrong config.** Switching toolchain/arch without `xmake f -c` leaves stale probe results. Re-configure on any toolchain change.
- **`has_cfuncs` passes but linker fails.** The probe links against the compiler's default libs; if you need an explicit `-lpthread`, declare it too: `has_cfuncs("pthread_create", {includes = "pthread.h", links = "pthread"})`.
- **Probes run at configure time, not build time.** Changing a header after `xmake f` won't invalidate the probe. Edit, then `xmake f -c`.
- **Snippet without `int main()`.** Most probes wrap your snippet for you, but some (`check_csnippets`) expect a self-contained compilation unit. Test with `xmake f -vD` if something looks off.
- **Using `is_plat` to fake a probe.** Hard-coding `if is_plat("linux") then add_cfuncs ... end` drifts on newer platforms. Probes are portable.

## When to branch out

- Option declaration mechanics → `xmake-options`
- Scripting / `import` of detect modules → `xmake-scripting`
- Generated config headers → `xmake-options` / `xmake-targets`
