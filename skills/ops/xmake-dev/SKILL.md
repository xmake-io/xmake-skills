---
name: xmake-dev
description: Use when hacking on Xmake itself — building the `xmake` binary from source, loading the local source tree so your changes take effect without reinstalling, and debugging either the Lua scripts under `xmake/` or the native C core under `core/src/xmake`.
---

# Developing Xmake Itself

This skill is for working on the [`xmake-io/xmake`](https://github.com/xmake-io/xmake) source tree — not for projects that *use* xmake. Xmake is split into two layers:

- **Lua layer** (`xmake/`) — most of xmake's behavior: actions, modules, rules, toolchains, sandbox. Editable without recompiling.
- **Native C core** (`core/src/xmake/`) — the Lua runtime and xmake extension C APIs (`os.*`, `io.*`, `process.*`, `fwatcher`, etc.), plus tbox. Changes here require rebuilding the `xmake` binary.

## 1. Get the source

```bash
git clone --recursive https://github.com/xmake-io/xmake.git
cd xmake
```

Use `--recursive` — xmake bundles tbox and other submodules under `core/src`.

## 2. Build the `xmake` binary

### Linux / macOS

```bash
./configure
make -j
```

Artifacts land in `core/build/xmake` (also symlinked via `build/xmake`).

### Windows

```bat
cd core
xmake
```

This uses an existing xmake to bootstrap the new one; the resulting binary is `core/build/xmake.exe`.

### Clean rebuild

```bash
make clean
./configure && make -j
```

On Windows: `cd core && xmake clean && xmake`.

## 3. Load the local source tree

So the `xmake` command in your shell points at the freshly built binary *and* loads Lua scripts from your checkout instead of the installed `~/.xmake`:

### Linux / macOS

```bash
source scripts/srcenv.profile
xmake --version
```

This script:

- aliases `xmake` to `./core/build/xmake` (or `./build/xmake`)
- sets `XMAKE_PROGRAM_FILE` to that binary
- sets `XMAKE_PROGRAM_DIR` to `./xmake` (the Lua script tree)
- aliases `xrepo` to `scripts/xrepo.sh`
- prints the resolved `programdir` / `programfile` so you can verify

### Windows

```bat
scripts\srcenv.bat
```

Opens a new shell pre-configured with `XMAKE_PROGRAM_DIR` and `XMAKE_PROGRAM_FILE`, and `core\build` on `PATH`.

### Manual alternative

If you do not want to source the script, set the env vars yourself:

```bash
export XMAKE_PROGRAM_DIR=/path/to/xmake/xmake      # the Lua tree
export XMAKE_PROGRAM_FILE=/path/to/xmake/core/build/xmake
```

Now every `xmake` invocation in any project uses your local checkout.

Verify with:

```bash
xmake l xmake.programdir
xmake l xmake.programfile
```

Both should point inside your checkout.

## 4. Debugging the Lua layer

This is the fast loop — **no rebuild required**. Once `srcenv.profile` is sourced, edit any file under `xmake/` and rerun `xmake` in a test project to see the effect.

### Print-style debugging

```lua
-- inside xmake/modules/... or xmake/actions/...
import("core.base.option")
print("DEBUG value = %s", some_table)
utils.vprint("only in -v mode: %s", val)     -- verbose-gated
cprint("${bright}colored output${clear}")
```

Run the user's command with diagnostics:

```bash
xmake -vD build            # -v verbose, -D diagnosis (full Lua tracebacks)
```

`-D` turns every error into a stack trace anchored in the Lua source files — point at those line numbers to find the failure.

### Inspecting state

```lua
import("core.base.option")
import("core.project.project")
import("core.project.config")

print(config.plat(), config.arch(), config.mode())
for name, target in pairs(project.targets()) do
    print("target %s -> %s", name, target:targetfile())
end
```

Run ad-hoc Lua with the full xmake module environment:

```bash
xmake l -vD path/to/script.lua
xmake l -c 'print(os.host(), os.arch())'
```

### Interactive REPL

```bash
xmake l
> import("core.base.option")
> print(os.host())
```

Great for exploring module APIs (`core.project.project`, `lib.detect.find_tool`, `core.tool.toolchain`, …).

### Running the xmake test suite

```bash
xmake l tests/run.lua                 # full suite
xmake l tests/run.lua [testname]      # one group
```

Tests live under `tests/` in the xmake checkout; each directory is a mini project exercising a feature.

## 5. Debugging the native C core

Native code is under `core/src/xmake/` (Lua C bindings), `core/src/sv/`, and the bundled `core/src/tbox`. Changes require a rebuild; the Lua tree is untouched.

### Build a debug binary

```bash
cd /path/to/xmake
./configure --mode=debug
make -j
```

(Or, equivalently, building `core/` directly: `cd core && xmake f -m debug && xmake`.)

The resulting `core/build/xmake` has symbols and assertions enabled. `srcenv.profile` still points at it, so `xmake` in any project now runs the debug binary.

### Run under a debugger

Linux:

```bash
gdb --args ./core/build/xmake build -vD
(gdb) break xm_os_cpdir            # any C symbol in core/src/xmake
(gdb) run
```

macOS:

```bash
lldb -- ./core/build/xmake build -vD
(lldb) b xm_os_cpdir
(lldb) run
```

Windows (MSVC): open `core/build/xmake.exe` in Visual Studio, set it as startup target, pass the command-line args, and set breakpoints in `core/src/xmake/*.c`.

### Where the C bindings live

- `core/src/xmake/os/` — `os.cp`, `os.mv`, `os.ls`, process spawn, …
- `core/src/xmake/io/` — `io.open`, `io.readable`, …
- `core/src/xmake/process/` — subprocess handling
- `core/src/xmake/fwatcher/` — file watching
- `core/src/xmake/string/`, `hash/`, `base64/`, `libc/` — misc helpers

Each C function is registered to Lua under a name like `xm_os_cpdir`. The full call path for a user-visible API is:

```
sandbox_os.cp()   (xmake/core/sandbox/modules/os.lua)
  -> os.cp()      (xmake/core/base/os.lua)
    -> os.cpdir() bound to xm_os_cpdir()  (core/src/xmake/os/cpdir.c)
      -> tb_directory_copy()              (core/src/tbox)
```

To trace a misbehaving `os.cp`, set a breakpoint in `xm_os_cpdir` (C) *and* drop a `print(...)` in `xmake/core/base/os.lua` (Lua) — you usually need both sides to understand what arguments crossed the boundary.

### Adding a new C binding

1. Add the C function in `core/src/xmake/<module>/<name>.c`.
2. Register it in that module's `xmake.h` / registration table.
3. Expose it from the matching `xmake/core/base/<module>.lua` wrapper.
4. Optionally add a sandbox wrapper under `xmake/core/sandbox/modules/<module>.lua`.
5. `make -j` to rebuild.
6. `xmake l -c 'print(os.mynewfn())'` to smoke test.

### Memory / leak checks

```bash
./configure --mode=debug
make -j
valgrind --leak-check=full ./core/build/xmake build -vD
# or on macOS
leaks --atExit -- ./core/build/xmake build -vD
```

tbox ships its own allocator hooks as well — look under `core/src/tbox/src/tbox/memory`.

## 6. Common commands while developing

```bash
xmake -vD                         # verbose + Lua tracebacks on error
xmake l path/to/script.lua        # run arbitrary Lua in xmake env
xmake l xmake.programdir          # which script tree is active
xmake l xmake.programfile         # which binary is active
xmake g --clean                   # clear global caches if something is stuck
xmake f -c                        # re-run configure on a test project
```

## 7. Pitfalls

- **Forgot to source `srcenv.profile`.** `xmake` will still resolve to the installed binary in `/usr/local/bin`, so your Lua edits in the checkout appear to do nothing. Confirm with `xmake l xmake.programdir`.
- **Edited a Lua file that lives under `~/.xmake/repositories/...`.** That is a clone of xmake-repo, not the xmake source — changes there affect package recipes, not xmake itself.
- **Native changes not taking effect.** Rebuild (`make -j`). If you only ran `source srcenv.profile`, that does not recompile — it just points at whatever binary you already have.
- **Stale configure cache.** After switching between `--mode=debug` and `--mode=release`, run `./configure --mode=<mode>` again before `make`.
- **Submodules out of date after a pull.** `git submodule update --init --recursive`.

## When to branch out

- Writing a user-facing plugin or custom task → `xmake-plugins`
- Authoring a rule or handling a new file extension → `xmake-rules`
- Working on a package recipe in xmake-repo → `xmake-repo-testing`
