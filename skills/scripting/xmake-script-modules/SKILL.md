---
name: xmake-script-modules
description: Use when writing custom logic in `xmake.lua` and you need the day-to-day script-domain APIs — `os.*` (exec/cp/mv/ls/getenv), `io.*` (readfile/writefile/lines), `path.*` (join/basename/translate), `table.*` (xmake-extended), `string.*`, `hash.*`, `print`/`cprint`/`vprint`, `try/catch`, `raise`. Reference of what's always available without `import()`.
---

# Built-in Script-Module Reference

Inside the script domain (`on_load`, `on_build`, `after_build`, `on_run`, `on_test`, task/rule/plugin bodies, imported `.lua` modules) xmake provides a set of **pre-imported** modules. No `import(...)` needed for any of these — they are injected into the sandbox.

For how to write modules and use `import()` itself, see `xmake-scripting`.

## `os` — filesystem, process, env

### Filesystem

```lua
os.exists("path")          -- true if file or dir exists
os.isfile("path")
os.isdir("path")
os.mkdir("a/b/c")          -- recursive
os.rmdir("a")
os.rm("file-or-glob")
os.cp("src", "dst")        -- file or dir; glob patterns supported
os.mv("src", "dst")
os.ls("dir")               -- returns list of basenames
os.files("src/**/*.cpp")   -- glob list of files
os.dirs("src/**")          -- glob list of dirs
os.filedirs("src/**")      -- glob files + dirs
os.emptydir("build")       -- delete all contents
```

Globs: `*` one level, `**` recursive, `?` single char. `(...)` in the pattern preserves directory structure during copy.

### Process

```lua
os.exec("cmd arg1 arg2")           -- raises on non-zero exit, streams output
os.execv(program, {args})          -- list form — no shell parsing (prefer this)
os.run("cmd")                      -- silent unless it fails
os.runv(program, {args})
os.vrun("cmd")                     -- silent; print only in -v mode
os.vrunv(program, {args})
os.iorun("cmd")                    -- returns stdout
os.iorunv(program, {args})         -- returns stdout, stderr
```

`*v` (list) forms are safer — they bypass shell quoting entirely. Prefer them whenever arguments come from variables.

### Environment / info

```lua
os.getenv("PATH")
os.setenv("FOO", "bar")            -- scope: current xmake process only
os.host()                          -- "macosx" | "linux" | "windows" | ...
os.arch()                          -- "x86_64" | "arm64" | ...
os.subhost()                       -- detailed host info
os.tmpdir()                        -- scratch dir
os.curdir()                        -- cwd
os.cd("path")                      -- change cwd
os.mtime("file")                   -- modification time
os.time()                          -- current unix time
os.date("%Y-%m-%d")
os.nuldev()                        -- /dev/null or NUL
os.argv()                          -- raw CLI args
```

## `io` — file I/O

```lua
io.readfile("path")                            -- whole file → string
io.writefile("path", "content")                -- overwrite
io.open("path", "r")                           -- Lua-style file handle
for line in io.lines("path") do ... end
io.gsub("path", "pattern", "replace")          -- in-place regex replace
```

`io.gsub` is the one-liner for string editing a file:

```lua
io.gsub("version.h", '#define VERSION "[^"]*"', '#define VERSION "1.2.3"')
```

## `path` — path manipulation

Cross-platform path handling (forward slash on Unix, backslash on Windows, auto-normalized):

```lua
path.join("a", "b", "c.txt")       -- "a/b/c.txt"
path.filename("a/b/c.txt")         -- "c.txt"
path.basename("a/b/c.txt")         -- "c" (no extension)
path.extension("c.txt")            -- ".txt"
path.directory("a/b/c.txt")        -- "a/b"
path.absolute("rel")               -- absolute form
path.relative("/a/b/c", "/a")      -- "b/c"
path.translate("a\\b/c")           -- normalize separators to host
path.is_absolute("/x")
```

Always prefer `path.join` over string concatenation — it handles separators per OS.

## `table` — xmake-extended

Standard Lua `table.*` plus:

```lua
table.contains(tbl, value)
table.concat(tbl, ", ")
table.insert(tbl, value)
table.remove(tbl, idx)
table.unique(tbl)                  -- dedupe in place
table.join(tbl1, tbl2, ...)        -- merge (returns new table)
table.copy(tbl)                    -- shallow copy
table.keys(tbl)
table.values(tbl)
table.unpack(tbl)                  -- Lua 5.1 / 5.3 unified
table.pack(...)
table.orderkeys(tbl)               -- stable key order
table.wrap(x)                      -- if x not table, wrap as {x}
```

## `string` — xmake-extended

```lua
string.split("a,b,c", ",")         -- {"a","b","c"}
string.trim("  x  ")               -- "x"
string.startswith("hello", "he")
string.endswith("hello", "lo")
string.format("%d: %s", 1, "ok")   -- standard
string.upper("abc") / string.lower("ABC")
```

## `hash` — cryptographic hashes

```lua
hash.md5("string-or-file-path")
hash.sha1("...")
hash.sha256("...")
hash.xxh64("...")                  -- fast non-crypto
```

Accepts either a string or a file path. Use for cache keys, change detection, integrity checks.

## Printing & logging

```lua
print("hello %s", name)            -- with newline
printf("hello %s", name)           -- no newline
cprint("${bright green}done${clear}")      -- with color tags
cprintf("${red}warning")           -- color, no newline
utils.vprint("only in -v mode")    -- gated by -v
utils.cprint("same as cprint")
raise("fatal: %s", reason)         -- throw, with format args
```

See `xmake-color-output` for the `${...}` tag reference.

## `try` / `catch` / `finally`

Xmake's structured error handling:

```lua
try {
    function ()
        os.exec("some-command")
    end,
    catch {
        function (errors)
            print("failed: %s", errors)
        end
    },
    finally {
        function ()
            print("cleanup")
        end
    }
}
```

All three blocks are optional (a plain `try { function () ... end }` is fine). The catch block receives the error message.

## Platform-specific helpers

```lua
if is_host("windows") then
    import("core.base.winos")            -- winos needs import
    -- winos.version(), winos.registry_*() etc.
end

linuxos.name()                           -- available without import
linuxos.version()
macos.version()
```

## `hashset` / `heap` / `list` — extended data structures

```lua
import("core.base.hashset")
local s = hashset.new()
s:insert("a")
s:contains("a")
```

Needs `import` — not pre-injected.

## What is **not** pre-imported

These need `import(...)`:

- `core.base.option`, `core.base.task`, `core.base.global`
- `core.project.project`, `core.project.config`
- `core.platform.platform`, `core.tool.toolchain`, `core.tool.compiler`, `core.tool.linker`
- `core.package.package`
- `lib.detect.find_tool`, `lib.detect.find_package`, `lib.detect.has_cfuncs`, etc.
- `net.http`, `devel.git`, `utils.archive`
- `core.base.hashset`, `core.base.heap`, `core.base.list`

See `xmake-scripting` for the import mechanics.

## Common patterns

### Copy build output to a dist dir

```lua
after_build(function (target)
    local dist = path.join(os.projectdir(), "dist")
    os.mkdir(dist)
    os.cp(target:targetfile(), dist)
end)
```

### Stamp a version header

```lua
before_build(function (target)
    local version = try {function () return os.iorun("git describe --tags") end} or "unknown"
    io.writefile("src/version.h",
        string.format('#define VERSION "%s"\n', version:trim()))
end)
```

### Walk all sources matching a pattern

```lua
on_load(function (target)
    for _, file in ipairs(os.files("src/**/generated_*.cpp")) do
        target:add("files", file)
    end
end)
```

### Run a tool and fail cleanly

```lua
before_build(function (target)
    import("lib.detect.find_tool")
    local protoc = assert(find_tool("protoc"), "protoc not found on PATH")
    os.vrunv(protoc.program, {"--cpp_out=gen", "proto/foo.proto"})
end)
```

## Pitfalls

- **Calling these in the description domain.** The script-domain modules are sandbox-restricted at description level; most mutating calls (`os.cp`, `os.exec`, `io.writefile`) raise. Move to `on_load` or later hooks.
- **Shell injection via `os.exec("cmd " .. var)`.** Prefer `os.execv` with a list — no shell parsing, no quoting surprises.
- **`os.iorun` in description domain.** Runs on every parse of `xmake.lua`. Move into `on_load` or a task.
- **Hard-coded path separators.** `"src\\\\main.cpp"` breaks on Unix. Use `path.join` or forward slashes (xmake normalizes forward slashes on all platforms).
- **Assuming Lua's stock `table.pack` / `table.unpack`.** Xmake's extended `table` shims these for Lua 5.1/5.3 consistency — safe to use, but don't mix with `unpack` (global).

## When to branch out

- Writing modules and using `import()` / native modules → `xmake-scripting`
- Color tag syntax for `cprint` → `xmake-color-output`
- Feature/capability probes (`has_cfuncs`, `check_cxxsnippets`) → `xmake-feature-check`
- Target instance APIs (`target:add`, `target:targetfile`) → `xmake-targets`
