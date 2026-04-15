---
name: xmake-remote-compilation
description: Use when compiling and running a project on a remote machine transparently — edit locally, build/run/debug on a Linux/macOS/Windows server. Covers `xmake service` (remote build server), multi-host client config, token/password auth, `--sync`/`--pull`, and the difference from distributed compilation.
---

# Xmake Remote Compilation

Remote compilation is "run the entire build + run + test on a remote machine, transparently". Different from distributed compilation (which offloads *individual* compile jobs to a cluster): remote compilation sends **one project** to **one server**, and every `xmake`/`xmake run`/`xmake test` command executes there.

Use it for:

- Editing on a laptop, building on a beefy Linux box.
- Cross-OS development (compile Windows binaries from macOS or vice versa) without ssh pain.
- Running targets on a device with the right hardware (GPU/NDK/Xcode).

Compared to ssh-based workflows, this doesn't suffer from flaky terminal sessions and integrates into any editor without IDE-specific remote-dev features.

## 1. Start the server

```bash
xmake service                       # foreground (starts all configured services)
xmake service -vD                   # verbose
xmake service --start               # daemonize
xmake service --stop
xmake service --restart
```

First run creates `~/.xmake/service/server.conf` and prints a token.

### Server config

```
{
    known_hosts = { },
    logfile     = "/home/ruki/.xmake/service/server/logs.txt",
    remote_build = {
        listen  = "0.0.0.0:9691",
        workdir = "/home/ruki/.xmake/service/server/remote_build"
    },
    tokens = { "e438d816c95958667747c318f1532c0f" }
}
```

- `listen` — host:port (default 9691).
- `workdir` — where projects are synced on the server.
- `tokens` — token allowlist.
- `known_hosts` — optional IP allowlist (harden exposed servers).

## 2. Client config — single host

`~/.xmake/service/client.conf`:

```
{
    remote_build = {
        connect = "192.168.56.110:9691",
        token   = "e438d816c95958667747c318f1532c0f"
    }
}
```

## 3. Client config — multiple hosts (v3.0.7+)

```
{
    remote_build = {
        hosts = {
            { name = "local",   connect = "127.0.0.1:9691",     token = "ab9dcb6f..." },
            { name = "windows", connect = "10.5.138.247:9691",  token = "0e052f8c..." },
            { name = "linux",   connect = "10.5.138.100:9691",  token = "7889e254..." }
        }
    }
}
```

Pick a host at connect time:

```bash
xmake service --connect                        # first host
xmake service --connect --host=windows         # by name
xmake service --connect --host=10.5.138.247    # by IP
xmake service --connect --host=10.5.138.247:9691
```

## 4. Authentication modes

- **Token (recommended).** Default. Generated automatically on first `xmake service` run. More of them: `xmake service --gen-token`.
- **Password.** `xmake service --add-user=<name>` on server, then `connect = "user@host:port"` on client. Prompts for password on each connect.
- **Trusted host.** Populate server `known_hosts` — clients outside the list are rejected even with a valid token.

## 5. Connect and work

```bash
cd myproject
xmake service --connect
# -> Scanning files ..
# -> Comparing 3 files ..
# -> Uploading files with 1372 bytes ..
# -> sync files ok!
```

First connect uploads the project to `<workdir>/<project-hash>` on the server. Subsequent syncs are incremental.

Once connected, every xmake command runs **on the server**:

```bash
xmake                    # build (on server)
xmake run                # run target (on server)
xmake run -d app         # debug (on server)
xmake test               # test (on server)
xmake f -m release       # configure (on server)
xmake -rv                # verbose rebuild (on server)
```

Output streams back to your terminal. Errors point at server paths (same as the project root, since xmake keeps the relative layout).

## 6. Syncing files

Code sync is automatic on build, but you can force it:

```bash
xmake service --sync     # push local changes now
```

Useful after a batch edit from a tool the file watcher didn't catch.

### Pull artifacts back (v2.7.1+)

```bash
xmake service --pull 'build/**' ./out         # glob + local dest
xmake service --pull build/linux/x86_64/release/myapp ./dist/
```

Use for grabbing the built library/executable after a remote compile, or fetching generated files (compile_commands.json, headers, coverage reports).

## 7. Disconnect / clean

```bash
xmake service --disconnect
xmake service --clean              # wipe server workdir for this project
xmake service --logs               # tail server logs
```

Disconnect is per-project — other projects stay connected.

## 8. Remote vs distributed — which?

| You want | Use |
| --- | --- |
| Build + run + debug on one beefy server | `xmake-remote-compilation` (this skill) |
| Pool compile jobs across a cluster while building locally | `xmake-distributed-compilation` |
| Share compiled objects across teammates/CI | `xmake-build-cache` (remote cache) |

They compose — a remote server can also be a distcc client/server, and a remote build can use the remote cache. Start with the simplest that solves your problem.

## Pitfalls

- **Remote paths in error messages confuse editors.** IDEs point at `/home/<server-user>/.xmake/service/...` paths. Either alias that path on your local machine or use `xmake service --pull` to bring artifacts/logs back.
- **Large project, slow first sync.** Xmake uploads the whole tree on first connect. Huge repos take minutes — use `.gitignore`-style filtering or trim build artifacts before connecting.
- **Config file format change in 2.6.6.** Pre-2.6.6 used a single `~/.xmake/service.conf`; current versions use separate `server.conf` / `client.conf` under `~/.xmake/service/`. If you're copying configs between old docs and new versions, watch for this.
- **Password prompt every connect.** Expected if you chose password auth — switch to token auth for frequent reconnects.
- **Running targets that need local resources.** Remote `xmake run` runs on the server — if the target expects a GPU/USB/filesystem that only exists locally, use remote for build only and pull the binary.

## When to branch out

- Multi-machine job distribution → `xmake-distributed-compilation`
- Shared compile cache → `xmake-build-cache`
- Auth/network issues debugging → `xmake-troubleshooting`
