---
name: xmake-distributed-compilation
description: Use when setting up or using xmake's built-in distributed compilation (distcc-equivalent) to offload compile jobs to a cluster of servers. Covers `xmake service --distcc`, server/client config, token auth, job control, combining with local/remote cache, and distributed cross-compilation (Android/iOS).
---

# Xmake Distributed Compilation

Xmake ships a built-in distributed compile service similar to distcc — but **fully cross-platform** (GCC, Clang, **and MSVC**) and tightly integrated with xmake's cache. Use it when a team or CI setup needs to pool compile resources.

## Architecture at a glance

```
         ┌──── distcc server (linux)  9693 ─┐
client ──┼──── distcc server (macos)  9693 ─┤──── jobs scheduled + compiled remotely
         └──── distcc server (win)    9693 ─┘
local compile ─── small/fast files stay local after preprocessing
local cache  ────── cache hits short-circuit everything
```

- Any machine running `xmake service --distcc` is a server.
- Clients list the servers in `client.conf`; xmake schedules jobs across them automatically.
- Server platforms can be mixed (Linux + macOS + Windows). Cross toolchains on the server do the actual work.

## 1. Start a server

```bash
xmake service --distcc                 # foreground
xmake service --distcc -vD             # foreground + verbose
xmake service --distcc --start         # daemonize
xmake service --distcc --stop
xmake service --distcc --restart
```

First run generates `~/.xmake/service/server.conf` and prints a token.

### Server config

```
{
    distcc_build = {
        listen  = "0.0.0.0:9693",
        workdir = "/home/ruki/.xmake/service/server/distcc_build"
    },
    known_hosts = { },
    logfile     = "/home/ruki/.xmake/service/server/logs.txt",
    tokens      = { "590234653af52e91b9e438ed860f1a2b" }
}
```

- `listen` — host:port.
- `workdir` — scratch dir for per-job files.
- `tokens` — whitelist. Generate more with `xmake service --gen-token`.
- `known_hosts` — optional IP allowlist.

### Cross toolchains on the server

If the server must cross-compile (e.g. Android from Linux, iOS from macOS), declare the toolchains in `server.conf` so the client doesn't need to ship them:

```
{
    distcc_build = {
        listen  = "0.0.0.0:9693",
        toolchains = {
            ndk = {
                ndk = "~/files/android-ndk-r26"
            },
            cross = {
                sdkdir = "~/files/arm-linux-musleabi-cross"
            }
        },
        workdir = "/home/ruki/.xmake/service/server/distcc_build"
    },
    tokens = { "..." }
}
```

Keys under `toolchains.*` match the xmake toolchain names (`ndk`, `cross`, `clang`, `gcc`, etc.) and map to `set_sdkdir`, `set_bindir`, `set_cross` fields.

## 2. Configure the client

`~/.xmake/service/client.conf`:

```
{
    distcc_build = {
        hosts = {
            {
                connect = "192.168.22.168:9693",
                token   = "590234653af52e91b9e438ed860f1a2b",
                njob    = 8
            },
            {
                connect = "192.168.22.169:9693",
                token   = "590234653af52e91b9e438ed860f1a2b",
                njob    = 4
            }
        }
    }
}
```

- `hosts` — array; jobs are load-balanced across entries.
- `njob = N` — max parallel jobs this particular server will handle. Omit to use the server's default.
- **Use token auth** for distcc. Password auth requires a prompt per connection, which is unworkable when you have multiple servers.

### Timeouts

```
{
    distcc_build = {
        connect_timeout = 5000,
        send_timeout    = 5000,
        recv_timeout    = 5000,
    }
}
```

Unit: milliseconds. On timeout the client automatically falls back to local compilation, so jobs don't hang.

## 3. Connect and build

```bash
cd myproject
xmake service --connect --distcc              # just distcc
xmake service --connect --distcc --ccache     # distcc + remote cache (recommended)
xmake                                          # compiles normally, jobs farmed out
```

Output during a build with distcc connected:

```
[ 93%]: cache compiling.release src/demo/network/unix_echo_client.c   ← local hit
[ 93%]: compiling.release        src/demo/network/ipv4.c              ← local
[ 93%]: distcc compiling.release src/demo/network/http.c              ← remote
[ 94%]: distcc compiling.release src/demo/math/fixed.c                ← remote
```

Three categories:
- **`cache compiling`** — served from the cache, no work done.
- **`compiling`** — compiled locally.
- **`distcc compiling`** — shipped to a server.

Xmake chooses automatically: small files after preprocessing often stay local (cheaper than round-tripping).

## 4. Job count math

Default total parallelism when distcc is connected:

```
n = ⌈3/2 × local_ncpu⌉          (default local job count)
c = server_count
d = per_server_default_njob
maxjobs = n + c × d
```

Override locally with `xmake -jN`. The `-j` flag does **not** override server-side job counts — those are set in the client config's `njob` field per host.

## 5. Cross-compilation projects

Once toolchains are declared on the server, client-side config is unchanged:

```bash
# Android from any client
xmake f -p android -a arm64-v8a --ndk=~/files/android-ndk-r26
xmake

# iOS (server must be macOS with Xcode)
xmake f -p iphoneos
xmake

# Custom cross
xmake f -p cross --sdk=/opt/arm-linux-musleabi-cross
xmake
```

The client does the configure locally, the server compiles each object.

## 6. Combining with caches

The ideal setup pairs all three:

```bash
xmake service --connect --distcc --ccache
```

Order of fallback per compile unit:

1. **Local cache** — instant hit, no network.
2. **Remote cache** — network hit, downloads precompiled object.
3. **Distributed compile** — farms out to a distcc server.
4. **Local compile** — fallback if everything above fails or times out.

See `xmake-build-cache` for the cache setup.

## 7. Management commands

```bash
xmake service --disconnect --distcc      # drop connection for current project
xmake service --clean --distcc           # clean this project's server scratch
xmake service --logs                     # tail server logs
xmake service --gen-token                # generate a new token
xmake service --add-user=ruki            # password-auth user (not recommended for distcc)
xmake service --rm-user=ruki
```

## Pitfalls

- **Password auth on multiple servers.** Don't. Use tokens — every host would prompt for a password per connection otherwise.
- **Network instability hangs the build.** Always set `connect_timeout` / `send_timeout` / `recv_timeout` so clients degrade to local on trouble.
- **Clients and servers must agree on compiler.** The compiler identity is part of the job key. Mixing gcc-11 clients with gcc-12 servers means no reuse and potentially subtle ABI differences.
- **`known_hosts` empty means open.** If you don't populate it, any host with a valid token can connect. For exposed networks, fill the allowlist.
- **Firewall on 9693.** Default distcc port — make sure it's open on every server machine.
- **Submodule / generated file sync.** Distcc ships preprocessed sources to the server, so generated headers *must* be produced locally before the compile step. If a rule's codegen hook only runs on the server, you'll get mysterious missing-header errors.

## When to branch out

- Caching compile results without distribution → `xmake-build-cache`
- Running the entire build on a single remote machine → `xmake-remote-compilation`
- Parallelism, unity build, PCH, other speedups → `xmake-build-optimization`
- Cross-compile setup the server needs → `xmake-cross-compilation`
