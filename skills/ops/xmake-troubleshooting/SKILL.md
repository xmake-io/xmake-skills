---
name: xmake-troubleshooting
description: Use when xmake is misbehaving — hangs/stuck, mysterious errors, slow configure/build, regressions after upgrade — and you need to diagnose it. Covers `-vD` verbose traces, `XMAKE_PROFILE=stuck/trace/perf:*` profiling env var, git bisect via `xmake l cli.bisect`, and debugging the Lua + C core. Do NOT use for everyday build errors — use the relevant per-topic skill first.
---

# Troubleshooting Xmake

When xmake itself (not your project) is misbehaving, work from the cheapest tool to the most invasive: verbose output → profiling env → bisect → debugger.

## 1. First: verbose and diagnosis output

```bash
xmake -v               # verbose: show every sub-command
xmake -vD              # verbose + diagnosis: full Lua tracebacks on error
xmake f -vD            # same but for configure
xmake l -vD path/to/script.lua
```

`-D` is the single most important flag when something goes wrong — any Lua error comes back with a full stack into `xmake/core/...`, which usually tells you exactly which module is unhappy.

Project cache can also mask bugs. If something seems sticky:

```bash
xmake f -c             # re-run configure, dropping cached state
xmake clean --all
rm -rf .xmake          # nuclear: drop the whole local cache
```

## 2. Stuck / hang diagnosis (`XMAKE_PROFILE=stuck`)

Xmake has a built-in stuck-process debugger. Set `XMAKE_PROFILE=stuck` and it will (a) log every subprocess it spawns and (b) on `Ctrl+C` print the current Lua stack traceback — so you can see exactly where it was blocked.

### Enable it

Linux / macOS:

```bash
XMAKE_PROFILE=stuck xmake              # one-shot
export XMAKE_PROFILE=stuck && xmake    # whole session
```

Windows PowerShell:

```powershell
$env:XMAKE_PROFILE="stuck"; xmake
```

Windows CMD:

```cmd
set XMAKE_PROFILE=stuck && xmake
```

### Reading the output

Subprocess trace (configure stuck probing a compiler):

```
$ XMAKE_PROFILE=stuck xmake f -c
<subprocess: sysctl>: /usr/sbin/sysctl -n machdep.cpu.vendor ...
checking for platform ... macosx
<subprocess: which>: which "xcrun -sdk macosx clang"
^C[xmake]: [engine]: stack traceback:
        @programdir/core/base/scheduler.lua:429: in function 'base/scheduler.co_suspend'
        .../lib/detect/find_program.lua:266: in function ...
        @programdir/modules/detect/tools/find_clang.lua:44: ...
```

This tells you:

- **Which subprocess was running** when it hung (`which "xcrun -sdk macosx clang"` here).
- **The Lua call stack** that led there — e.g. `find_clang` → `find_program` → `scheduler.co_suspend`, so xmake was waiting on a spawned process that never returned.

Typical root causes this surfaces:

- A compiler probe running against a broken/hung binary (e.g. a corrupt `clang`, a misconfigured wrapper).
- A package download from a slow/unreachable URL.
- A lock held by another `xmake` process (check with `ps`/Task Manager).

Report the trace verbatim when filing issues — it is far more useful than "xmake is stuck".

## 3. Performance / profiling

Same env var, different values. All enable per-phase timing and print a report at the end of the run.

| `XMAKE_PROFILE=...` | What it measures |
| --- | --- |
| `stuck` | Subprocess trace + Ctrl+C backtrace (see above) |
| `trace` | Trace every sub-command xmake runs |
| `perf:call` | Per-Lua-function call count and total time |
| `perf:tag` | Time spent in named profiler tags (e.g. `compile`, `link`) |
| `perf:process` | Time spent in spawned subprocesses |

Examples:

```bash
XMAKE_PROFILE=perf:call    xmake                # hot Lua functions
XMAKE_PROFILE=perf:tag     xmake                # compile/link phase timing
XMAKE_PROFILE=perf:process xmake -j1            # which child processes dominate
XMAKE_PROFILE=trace        xmake f -c           # log everything configure does
```

### Typical performance findings

- **`xmake f -c` is slow** → `perf:process` usually shows tool-detection probes dominating. Look for custom `on_check` / `find_*` modules that re-probe on every configure.
- **Incremental builds are slow** → `perf:tag` shows whether time is in `compile`, `link`, or elsewhere (dep-scan, codegen rules).
- **`xmake.lua` parsing is slow** → `perf:call` reveals expensive work done in the description phase. Move it to `on_load` / `on_config`.

Also useful:

```bash
xmake -j N              # parallel jobs
xmake build --jobs=N
xmake build --verbose   # show actual compile invocations
```

## 4. Bisecting a regression (`xmake l cli.bisect`)

When a feature stopped working between two xmake versions (or two commits of your `xmake.lua`), use xmake's built-in bisect wrapper. It runs `git bisect` under the hood and runs your test command at each step.

### Basic form

```bash
xmake l cli.bisect \
  -g <good>                        \
  -b <bad>                         \
  --gitdir=<path-to-xmake-checkout> \
  -c "<test command>"
```

- `-g, --good` — known-good tag or commit
- `-b, --bad` — known-bad tag or commit
- `--gitdir=` — path to the xmake source checkout to bisect
- `-c, --commands` — shell commands to run at each step; **multiple commands separated by `;`**. Non-zero exit → bad commit.
- `-s, --script` — Lua script instead of `-c`
- `--` — everything after is a raw command to execute

### Example: regression between v2.9.1 and v2.9.2

```bash
xmake l cli.bisect \
  -g v2.9.1 -b v2.9.2 \
  --gitdir=/path/to/xmake \
  -c "xrepo remove --all -y; xmake f -a arm64 -cvD -y"
```

Output on completion:

```
aa278bc7fc0b723a315120bb95531991b7939229 is the first bad commit
    improve to system/find_package
    xmake/modules/package/manager/system/find_package.lua | 16 +++++++++++++---
```

You now have the exact commit, author, and diff — paste into the bug report.

### Using a Lua test script (for non-trivial checks)

```bash
xmake l cli.bisect -s /tmp/test.lua -g v2.9.1 -b v2.9.2 --gitdir=/path/to/xmake
```

```lua
-- /tmp/test.lua
function main()
    os.exec("xrepo remove --all -y")
    os.exec("xmake f -a arm64 -cvD -y")
    os.exec("xmake build")
    -- for custom checks, use raise() to mark the commit as bad
    local output = os.iorun("xmake run hello")
    if not output:find("expected output") then
        raise("test output mismatch")
    end
end
```

Semantics: `os.exec` failures automatically mark the commit bad. Use `raise(...)` for custom assertions (output matching, file existence, …).

### Raw-command form

```bash
xmake l cli.bisect -g 90846dd -b ddb86e4 --gitdir=/path/to/xmake -- xmake -rv
```

Everything after `--` is executed verbatim. Good for one-liners.

### Tips

- Keep each test step **fast** (seconds, not minutes). Bisect runs it `log2(N)` times.
- Start from a clean state in the test command (`xrepo remove --all -y; xmake f -c`) so cached state from a previous step does not contaminate the next.
- Use a minimal reproduction project, not your real repo. Huge projects make every bisect step slow.
- If the good/bad ends are across a tbox submodule bump, run `git submodule update --init --recursive` inside the test script.

## 5. Debugging xmake source itself

If bisect points at a specific change and you want to dig in further, switch to source-level debugging. The full flow (build from source, `srcenv.profile`, editing Lua live, GDB/LLDB on the C core) has its own skill — see **`xmake-dev`**. A one-paragraph summary:

```bash
cd xmake && ./configure && make -j
source scripts/srcenv.profile
xmake l xmake.programdir          # verify you are running the local tree
xmake -vD                         # reproduce the issue; errors now point into your checkout
```

From there, edit `xmake/...` Lua files directly (no rebuild needed) and drop `print`/`utils.vprint`/`cprint` where the traceback points.

For native-side issues (`core/src/xmake/*.c`), `./configure --mode=debug && make -j`, then `gdb --args core/build/xmake ...` — see `xmake-dev`.

### EmmyLua debugger (step-through for Lua)

For interactive step-through of the Lua layer inside VS Code:

```bash
xrepo env -b emmylua_debugger -- xmake build
```

Then in VS Code (with the EmmyLua extension), open the xmake Lua source, go to *Run and Debug* → *Emmylua New Debug* to attach. The first breakpoint hits inside `debugger:_start_emmylua_debugger`; step out to land in `main.lua`. Swap `xmake build` for any other subcommand to debug it.

## 6. Filing a useful bug report

When escalating to maintainers, include:

1. **Xmake version**: `xmake --version`
2. **Platform and host**: `xmake l 'print(os.host(), os.arch(), os.subhost())'`
3. **Minimal `xmake.lua`** that reproduces the issue
4. **Full `-vD` output** (or at least the traceback)
5. If hang → **`XMAKE_PROFILE=stuck` subprocess trace + Ctrl+C backtrace**
6. If regression → **the commit from `xmake l cli.bisect`**
7. What you already tried (`xmake f -c`, `xmake clean --all`, deleting `.xmake/`)

## Diagnostic decision tree

```
Something is wrong with xmake
│
├── It just errors
│     └─ xmake -vD → read the Lua traceback
│
├── It hangs
│     └─ XMAKE_PROFILE=stuck xmake ... → Ctrl+C → read trace
│
├── It's slow
│     ├─ XMAKE_PROFILE=perf:process  → which subprocesses dominate?
│     ├─ XMAKE_PROFILE=perf:tag      → compile vs link vs other?
│     └─ XMAKE_PROFILE=perf:call     → expensive Lua functions?
│
├── It used to work, now it doesn't
│     └─ xmake l cli.bisect -g <old> -b <new> --gitdir=... -c "..."
│
└── I need to step through the source
      └─ see the `xmake-dev` skill (build + srcenv + gdb/emmylua)
```

## When to branch out

- Building xmake from source / Lua + C debugging in depth → `xmake-dev`
- Package install fails specifically → `xmake-packages`, `xmake-repo-testing`
- Toolchain-detection hangs → `xmake-toolchains`
