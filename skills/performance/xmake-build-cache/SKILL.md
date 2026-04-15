---
name: xmake-build-cache
description: Use when configuring or debugging xmake's local build cache (built-in, replaces ccache) or the remote compile cache service (shared across machines via `xmake service --ccache`). Covers enabling/disabling, fallbacks to external ccache, remote cache server/client setup, and cache cleaning.
---

# Xmake Build Cache (Local & Remote)

Xmake has a built-in compile cache that speeds up rebuilds by reusing object files. Two flavors:

- **Local cache** — on by default, per-machine, cross-platform (GCC/Clang/MSVC). Replaces ccache for most users.
- **Remote cache** — a shared server that machines can push/pull objects from. Like Mozilla sccache but cross-platform.

Both use the same `--ccache` name on the CLI — `--ccache=` is xmake-speak for "C/C++ build cache", not the ccache tool specifically.

## 1. Local cache (default)

Enabled automatically. No setup required. Since v2.6.6, xmake uses its own built-in local cache; before that it shelled out to the system `ccache`.

### Toggle

```bash
xmake f --ccache=n           # disable for this project
xmake f --ccache=y           # re-enable
```

### Use external ccache instead

If you specifically want the third-party `ccache` tool (e.g. for CI cache sharing with non-xmake builds):

```bash
xmake f --ccache=n --cxx="ccache g++" --cc="ccache gcc"
xmake
```

Note the `--ccache=n` — disables xmake's built-in cache so the two don't stack.

### Benefits of the built-in cache over external ccache

- **Cross-platform.** Works on Windows/MSVC; ccache doesn't.
- **No subprocess overhead.** Xmake does the lookup in-process — no per-file `ccache` fork.
- **Integrated state.** Xmake already tracks dependencies, so cache keys are tighter.

### Cache location

- Local cache directory: `~/.xmake/cache/` (cross-project shared).
- Clean it:
  ```bash
  xmake g --clean                # nuclear: wipe global cache
  ```

## 2. Remote cache service

Shared cache server. First developer to compile uploads objects; everyone else downloads them. Big win for CI + team setups.

### Start the server

```bash
xmake service --ccache                     # foreground
xmake service --ccache -vD                 # verbose
xmake service --ccache --start             # daemonize
xmake service --ccache --stop
xmake service --ccache --restart
```

First run creates `~/.xmake/service/server.conf` and prints a generated token.

### Server config (`~/.xmake/service/server.conf`)

```
{
    remote_cache = {
        listen = "0.0.0.0:9692",
        workdir = "/Users/ruki/.xmake/service/server/remote_cache"
    },
    known_hosts = { },
    logfile = "/Users/ruki/.xmake/service/server/logs.txt",
    tokens = {
        "590234653af52e91b9e438ed860f1a2b"
    }
}
```

- `listen` — host:port to bind.
- `workdir` — cache storage on the server.
- `tokens` — list of allowed client tokens. Generate more with `xmake service --gen-token`.
- `known_hosts` — optional IP allowlist (harden access).

### Client config (`~/.xmake/service/client.conf`)

```
{
    remote_cache = {
        connect = "cache.internal:9692",
        token   = "590234653af52e91b9e438ed860f1a2b"
    }
}
```

### Connect and use

```bash
cd myproject
xmake service --connect --ccache          # just the cache service
xmake service --connect --distcc --ccache # combine with distributed compile
xmake                                      # builds as usual, cache hits are transparent
```

Cache hits show up in output as the normal "cache compiling" line — xmake pulls from the remote server instead of compiling.

### Disconnect / clean

```bash
xmake service --disconnect --ccache
xmake service --clean --ccache             # wipe this project's cache on the server
xmake clean --all                          # (connected) also cleans remote cache
```

### Timeouts

If the server network is flaky, add timeouts so clients fall back to local compilation instead of hanging:

```
{
    remote_cache = {
        connect_timeout = 5000,
        send_timeout    = 5000,
        recv_timeout    = 5000,
    }
}
```

Unit: milliseconds. Works on both server and client sides.

## 3. Inspecting cache behavior

```bash
xmake -v                         # verbose: shows `cache compiling` vs normal lines
xmake --rebuild                  # force full rebuild, bypass cache
xmake -rv                        # rebuild verbose
```

In the build log:

- `cache compiling` — served from cache (local or remote).
- `compiling` (plain) — cache miss, actually compiled.
- `distcc compiling` — delegated to a distcc server (see `xmake-distributed-compilation`).

## Pitfalls

- **`--ccache=n` but cache still active.** You disabled the xmake built-in, but if `CXX=ccache g++` is in the env, ccache is still interposing. `unset CXX CC` or point at the raw compiler.
- **Remote cache hanging the build.** Add `connect_timeout` / `send_timeout` / `recv_timeout` so the client degrades to local on network trouble.
- **Token mismatch.** Every server restart does **not** regenerate tokens, but the first run does. If you nuke `~/.xmake/service/`, all clients need the new token.
- **Cache is project-scoped on the server.** Cleaning "the cache" cleans only the current project (by workdir hash). `xmake service --clean --ccache` inside a different project only affects that project.
- **Compiler version must match.** Cache keys include the compiler identity. A cache filled by gcc-11 on one box won't hit for gcc-12 on another — that's correct, not a bug.

## When to branch out

- Distributing the actual compile jobs across machines (not just caching) → `xmake-distributed-compilation`
- Remotely compiling a project on a single server (no distribution) → `xmake-remote-compilation`
- Every other build-speed knob (parallelism, unity build, PCH, etc.) → `xmake-build-optimization`
