---
name: xrepo-env
description: Use when the user wants a virtual environment of C/C++ libraries and CLI tools managed by xrepo — dropping into a shell with python/luajit/cmake/etc. on PATH, pinning per-project tool versions via `xmake.lua`, or registering named global environments. Conceptually similar to conda/venv but for C/C++ and native CLI tools.
---

# Xrepo Virtual Environments

`xrepo env` gives you a **virtual environment for native tools and libraries**. You describe a set of packages, xrepo installs them, and then sets up `PATH`, `LD_LIBRARY_PATH`, `DYLD_LIBRARY_PATH`, `PKG_CONFIG_PATH`, and any package-specific env vars so those tools "just work" inside the env.

Think conda/venv, but for C/C++ dependencies and CLI tools (cmake, ninja, python, luajit, protoc, …), sourced from [xmake-repo](https://github.com/xmake-io/xmake-repo) or any registered private repo.

There are three ways to use it, ordered from most ad-hoc to most persistent.

## 1. Ad-hoc environments (no config file)

Pass a comma-separated package list to `-b/--bind` and `xrepo env` auto-installs and activates them.

### Interactive shell

```bash
xrepo env -b "python 3.x,luajit,cmake" shell
[python,luajit,cmake] $ python --version
Python 3.10.6
[python,luajit,cmake] $ cmake --version
cmake version 3.25.3
[python,luajit,cmake] $ xrepo env quit
$
```

- Missing packages are installed on demand (first entry may be slow; subsequent entries are instant).
- The prompt is prefixed with the active package list so you know where you are.
- `xrepo env quit` (or just `exit`) leaves the subshell.

### One-shot command

Skip the subshell and run a single command inside the env:

```bash
xrepo env -b "python 3.x" python script.py
xrepo env -b "cmake,ninja" cmake -B build -G Ninja
xrepo env -b "protobuf" protoc --version
xrepo env -b "llvm 18.x" clang++ hello.cpp -o hello
```

Great for scripting / CI — no subshell juggling.

### Version specs and configs

Same syntax as `add_requires`:

```bash
xrepo env -b "python 3.11.x,cmake >=3.25" shell
xrepo env -b "boost" -f "regex=true" shell         # package configs
xrepo env -b "llvm" -m debug shell                 # debug build
```

## 2. Project-level environments (`./xmake.lua`)

Drop an `xmake.lua` in a directory — `xrepo env` in that directory reads it automatically. This is the right choice when a project needs a reproducible set of tools per working tree, checked into git.

```lua
-- ./xmake.lua
add_requires("zlib 1.2.11")
add_requires("python 3.x", "luajit")
add_requires("cmake 3.27.x", "ninja")

-- optional: also load a toolchain env
set_toolchains("msvc")     -- or "clang", "gcc-11", "llvm-mingw", ...
```

```bash
cd myproject
xrepo env shell            # activates everything in ./xmake.lua
xrepo env cmake --version  # one-shot
```

This is the xmake equivalent of a `requirements.txt` / `environment.yml` that also pins your compiler.

## 3. Named / registered environments

Register an env file globally and refer to it by name from anywhere. Use this for long-lived environments you switch between — "my embedded toolchain", "the CUDA build env", etc.

### Register, list, remove

```bash
xrepo env --add /tmp/base.lua            # register file as env "base"
xrepo env --list                         # show all registered envs
xrepo env --remove base
```

Registered envs live under `~/.xmake/envs/`.

Example `base.lua`:

```lua
add_requires("python 3.11.x")
add_requires("cmake", "ninja", "llvm 18.x")
```

### Activate by name

```bash
xrepo env -b base shell                  # interactive
xrepo env -b base python --version       # one-shot
xrepo env -b base cmake -B build
```

### Multiple named envs

```bash
xrepo env --add envs/cuda.lua     cuda
xrepo env --add envs/embedded.lua embedded
xrepo env -b cuda      nvcc --version
xrepo env -b embedded  arm-none-eabi-gcc --version
```

## Inspecting what an env sets

Before activating, see the exact environment variables that would apply:

```bash
xrepo env --show luajit
xrepo env --show -b base
xrepo env --show -b "python 3.x,cmake"
```

Example output:

```lua
{
    PATH = "/home/ruki/.xmake/packages/l/luajit/.../bin:...",
    LD_LIBRARY_PATH = "/home/ruki/.xmake/packages/l/luajit/.../lib",
    PKG_CONFIG_PATH = "...",
    XMAKE_PROGRAM_DIR = "...",
    ...
}
```

Use this to debug "why isn't `python` the one I expect" — the first `PATH` entry wins.

## CI usage

`xrepo env` one-shot mode is perfect for CI — no shell management, no activation scripts:

```yaml
# GitHub Actions example
- run: curl -fsSL https://xmake.io/shget.text | bash
- run: xrepo env -b "cmake 3.27.x,ninja,llvm 18.x" cmake -B build -G Ninja
- run: xrepo env -b "cmake 3.27.x,ninja,llvm 18.x" cmake --build build
```

Or commit an `envs/ci.lua` and reference it:

```bash
xrepo env --add envs/ci.lua ci
xrepo env -b ci bash ./scripts/build.sh
```

## Combining with toolchains

`set_toolchains(...)` inside the env file loads that compiler's environment too — VS vars on Windows, Xcode SDK on macOS, a cross SDK on Linux. This lets a single `xrepo env shell` give you both the right `cmake`/`ninja` *and* the right compiler.

```lua
add_requires("cmake", "ninja")
set_toolchains("msvc", {vs = "2022"})
```

```bash
xrepo env shell
> cl /?          # MSVC on PATH, env loaded
> cmake -G Ninja ..
```

## Typical workflows

- **"I need cmake 3.27 for this one-off task"** → `xrepo env -b "cmake 3.27.x" cmake ...`
- **"I want a reproducible dev shell for this repo"** → check in `xmake.lua` with `add_requires`, use `xrepo env shell`
- **"I juggle several toolchains/SDKs"** → register each as a named env, switch with `-b <name>`
- **"CI needs a pinned python + cmake + ninja"** → one-shot `xrepo env -b "..." <cmd>` per step, or a committed `envs/ci.lua`
- **"What does this env actually do to my PATH?"** → `xrepo env --show -b ...`

## Pitfalls

- **Quoting matters.** `xrepo env -b python 3.x shell` splits `python` and `3.x` as separate args. Always quote: `-b "python 3.x"`. Same for comma lists: `-b "cmake,ninja"`.
- **Env doesn't update after editing `xmake.lua`.** Re-enter the env — `xrepo env` reads the file at activation time; an already-running subshell won't pick up edits.
- **Tool is still the system one.** Check with `which cmake` and `xrepo env --show`; if the registered env's `PATH` prefix isn't there, the env failed to install that package — rerun with `-vD`.
- **Named envs are file references, not snapshots.** `xrepo env --add foo.lua` stores a pointer; if you move or delete `foo.lua`, the env breaks. Keep env files somewhere stable (e.g. inside the repo).
- **`quit` vs `exit`.** Both leave the subshell; `xrepo env quit` is just a friendlier alias. Nested envs are allowed — `quit` pops one level.

## When to branch out

- Installing / fetching / exporting packages outside an env → `xrepo-cli`
- Using packages from an `xmake.lua` build (not just tools on PATH) → `xmake-packages`
- Toolchain setup referenced via `set_toolchains(...)` → `xmake-toolchains`
