---
name: xmake-scripting
description: Use when writing custom Lua script logic inside `xmake.lua` — `on_load`/`on_build`/`after_build` hooks, `import()`, extracting script files under a `modules/` dir, importing xmake core modules (`core.base.option`, `lib.detect.find_tool`, …), or building native C/C++ Lua modules via `add_rules("module.shared" / "module.binary")` with `add_moduledirs`.
---

# Xmake Scripting, `import`, and Native Modules

`xmake.lua` is Lua, but "writing Lua" in xmake means working inside the **script domain** — hook bodies like `on_load`, `on_build`, `after_build`, `on_install`, `on_run`, `on_test`, or standalone modules loaded through `import()`. This skill covers how to write, organize, and load that code, including **native (C/C++) Lua modules**.

If you are looking for the description/script domain *boundary* and style rules, see `xmake-style`. This skill picks up once you are inside a script.

## 1. Where scripts live

Three idiomatic forms, from inline to fully external:

### Inline (short logic)

```lua
target("app")
    set_kind("binary")
    add_files("src/*.cpp")
    after_build(function (target)
        os.cp(target:targetfile(), "$(buildir)/dist/")
    end)
```

Fine for anything under ~15 lines.

### External file under `modules/`

```lua
target("app")
    on_load("modules.app.load")
    on_install("modules.app.install")
```

With a tree:

```
xmake.lua
modules/
  app/
    load.lua
    install.lua
```

Each external script file uses the **module convention** (see below) — no `return`, a `main` function is the entry point.

### Standalone reusable module imported with `import()`

```
xmake.lua
scripts/
  codegen.lua
  utils.lua
```

```lua
-- inside some on_load / on_build
import("scripts.codegen")
codegen.generate(target)
```

## 2. The module convention

Xmake does **not** use Lua's native `require` / `return` pattern. Module files follow xmake's own convention:

```lua
-- scripts/foo.lua

-- private: prefixed with underscore, not exported
function _helper(x)
    return x * 2
end

-- public: any top-level function without leading underscore
function add(a, b)
    return _helper(a) + b
end

function greet(name)
    print("hello, %s!", name)
end

-- optional: `main` is the module's callable entry point
function main(a, b)
    return add(a, b)
end
```

Imported:

```lua
import("scripts.foo")

foo.add(1, 2)           -- public function
foo.greet("ruki")       -- public function
foo(1, 2)               -- calls main()
-- foo._helper(1)       -- ERROR: underscore-prefixed is private
```

Equivalent one-shot: `import("scripts.foo")(1, 2)`.

Rules:

- No `return` at the end of the file.
- Top-level functions **without** `_` prefix are public.
- Top-level functions **with** `_` prefix are private.
- `main(...)` is optional; if present, calling the imported table invokes it.

## 3. `import()` — loading modules

```lua
import("core.base.option")            -- built-in xmake module
import("core.project.project")
import("lib.detect.find_tool")
import("scripts.foo")                 -- local module at ./scripts/foo.lua
```

Resolution order:

1. Current script directory (relative to the file doing the import).
2. xmake's built-in extension class libraries under `$(programdir)/modules/` and `$(programdir)/core/`.
3. Any directories added with `add_moduledirs(...)`.

### Options

```lua
import("core.platform.platform", {alias = "p"})            -- rename to avoid conflicts
import("scripts.foo",             {rootdir = "/abs/path"}) -- import from elsewhere
import("scripts.foo",             {try = true})            -- return nil on failure
import("scripts.foo",             {anonymous = true})      -- don't pollute scope
import("scripts.base",            {inherit = true})        -- merge into current module
```

- `alias` — bind the module to a different local name. Critical when two modules share a base name.
- `rootdir` — import from an absolute directory, useful for shared tool libraries.
- `try` — don't throw if the module doesn't exist; return `nil` instead.
- `anonymous` — don't create a top-level local; return the import object and let the caller bind it.
- `inherit` — copy all public functions into the calling module (module-level inheritance).

## 4. Frequently imported xmake core modules

| Module | What it gives you |
| --- | --- |
| `core.base.option` | CLI option parsing (`option.get("name")`) inside tasks/plugins |
| `core.base.task` | Run another task programmatically (`task.run("build")`) |
| `core.base.global` | Global config (`~/.xmake/xmake.conf` values) |
| `core.project.project` | All targets in the project: `project.targets()`, `project.target("name")` |
| `core.project.config` | Configured plat/arch/mode/buildir |
| `core.platform.platform` | Host/target platform info, `platform.plats()` |
| `core.tool.toolchain` | Toolchain objects |
| `core.tool.compiler` / `core.tool.linker` | Invoke a compiler/linker programmatically |
| `core.package.package` | Package instances, `package:installdir()`, etc. |
| `lib.detect.find_tool` | Probe for an executable on PATH / SDK |
| `lib.detect.find_package` | Probe for a library |
| `lib.detect.find_program` | Lower-level program probe |
| `net.http` / `net.fasturl` | HTTP fetching with xmake's downloader |
| `devel.git` | Git operations (clone, ls-remote) |
| `utils.archive` | Zip/tar extraction |

Example — find a tool and shell out through it:

```lua
on_build(function (target)
    import("lib.detect.find_tool")
    import("core.base.option")

    local protoc = assert(find_tool("protoc"), "protoc not found")
    os.vrunv(protoc.program, {"--cpp_out=" .. target:autogendir(), "proto/msg.proto"})
end)
```

## 5. Target/config APIs from inside a hook

Inside `on_load(function (target) ... end)` the `target` argument gives you a target instance:

```lua
target:name()
target:kind()
target:targetfile()          -- full path of the built file
target:sourcefiles()         -- list of source files
target:installdir()
target:pkg("fmt")            -- get a package instance bound to the target
target:add("links", "pthread")
target:add("defines", "DEBUG", {public = true})
target:set("kind", "binary")
target:get("includedirs")
target:values("cxflags")
```

`target:add` / `target:set` / `target:get` are the script-domain equivalents of `add_*` / `set_*` / the raw config.

## 6. Built-in modules you can use in scripts

All of these are pre-imported in the script domain — no `import(...)` needed:

- **`os`** — cross-platform `os.cp`, `os.mv`, `os.rm`, `os.mkdir`, `os.exists`, `os.exec`, `os.vrunv`, `os.iorunv`, `os.getenv`, `os.mtime`, `os.tmpdir`, `os.host`, `os.arch`.
- **`io`** — `io.readfile`, `io.writefile`, `io.open`, `io.lines`, `io.gsub`.
- **`path`** — `path.join`, `path.filename`, `path.basename`, `path.extension`, `path.directory`, `path.absolute`, `path.relative`, `path.translate`.
- **`table`** — xmake-extended: `table.concat`, `table.contains`, `table.insert`, `table.unique`, `table.join`, `table.copy`.
- **`string`** — xmake-extended: `string.split`, `string.trim`, `string.startswith`, `string.endswith`.
- **`hash`** — `hash.md5`, `hash.sha1`, `hash.sha256`, `hash.xxh64`.
- **`print` / `cprint` / `vprint` / `printf` / `cprintf`** — xmake's logging. `cprint` supports `${color}` tags; `vprint` only prints in `-v` mode.
- **`raise`** — throw an error. Inside `cli.bisect` or `on_test`, a raised error marks the step bad.
- **`try ... catch ... finally`** — structured error handling, xmake's Lua-level equivalent.
- **`winos`** / **`linuxos`** / **`macos`** — host-specific helpers.

Full reference lives under `xmake/core/base/` and `xmake/core/sandbox/modules/` in the xmake source.

## 7. Shell-outs: pick the right `os.*`

| Call | Return | Use for |
| --- | --- | --- |
| `os.exec("cmd arg1 arg2")` | void (throws on fail) | Run a command, stream output to the user |
| `os.execv(program, {args...})` | void | Same, but args passed as a list (no shell parsing) |
| `os.vrunv(program, {args...})` | void | Like `execv` but only prints command in `-v` mode |
| `os.run("cmd ...")` | void | Run silently unless it fails |
| `os.runv(program, {args...})` | void | Silent, list-form args |
| `os.iorun("cmd ...")` | `stdout` | Capture stdout |
| `os.iorunv(program, {args...})` | `stdout, stderr` | Capture both |

Prefer `*v` forms (list args) over string forms — no shell quoting surprises. Prefer `vrunv` inside build hooks so `-v` controls visibility.

## 8. Custom tasks — callable from the CLI

Wrap a module into a `task` and it becomes `xmake <taskname>`:

```lua
task("hello")
    set_menu {
        usage = "xmake hello [options]",
        description = "Say hello",
        options = {
            {"n", "name", "kv", "world", "name to greet"}
        }
    }
    on_run(function ()
        import("core.base.option")
        cprint("${bright green}hello, %s!${clear}", option.get("name"))
    end)
```

```bash
xmake hello --name=ruki
```

Option kind: `"k"` (flag), `"kv"` (key-value), `"vs"` (values list). See `xmake-plugins` for more.

## 9. Native modules (C/C++ Lua modules)

When Lua is too slow (heavy hash/parse/math), or when you want to reuse an existing C library, drop into a **native module**. Xmake builds it automatically and loads it via `import()` just like a Lua module.

Two flavors:

- **Shared (`module.shared`)** — a `.so`/`.dll`/`.dylib` loaded into the xmake process. Fastest; exposes Lua C API.
- **Binary (`module.binary`)** — an executable spawned per call. Simpler, cross-platform, but slower.

### Shared native module

Directory layout:

```
xmake.lua
modules/
  foo/
    xmake.lua            # builds the module
    foo.c                # native source
```

Module xmake.lua:

```lua
-- modules/foo/xmake.lua
add_rules("mode.debug", "mode.release")

target("foo")
    add_rules("module.shared")
    add_files("foo.c")
```

Native source:

```c
// modules/foo/foo.c
#include <xmi.h>   // xmake's lua include shim — prefer over <lua.h>

static int c_add(lua_State* lua) {
    int a = lua_tointeger(lua, 1);
    int b = lua_tointeger(lua, 2);
    lua_pushinteger(lua, a + b);
    return 1;
}

static int c_sub(lua_State* lua) {
    int a = lua_tointeger(lua, 1);
    int b = lua_tointeger(lua, 2);
    lua_pushinteger(lua, a - b);
    return 1;
}

int luaopen(foo, lua_State* lua) {
    static const luaL_Reg funcs[] = {
        {"add", c_add},
        {"sub", c_sub},
        {NULL, NULL}
    };
    lua_newtable(lua);
    luaL_setfuncs(lua, funcs, 0);
    return 1;
}
```

Key points:

- Include `xmi.h`, not `lua.h`/`luaconf.h` directly — it papers over Lua vs LuaJIT differences.
- Xmake's main binary already exports the full Lua C API — **no Lua dependency needed** in your module.
- `luaopen(<name>, lua_State*)` is the entry point; xmake calls it when the module is imported.

### Binary native module

```
modules/bar/
  xmake.lua
  bar.cpp
```

```cpp
// modules/bar/bar.cpp
#include <cstdio>
#include <cstdlib>
int main(int argc, char** argv) {
    int a = atoi(argv[1]);
    int b = atoi(argv[2]);
    printf("%d", a + b);
    return 0;
}
```

```lua
-- modules/bar/xmake.lua
target("add")
    add_rules("module.binary")
    add_files("bar.cpp")
```

The module protocol: xmake spawns the binary with the args, reads stdout for the return value. No Lua API involved.

### Consuming the module

```lua
-- ./xmake.lua
add_moduledirs("modules")           -- tell xmake where to look

target("app")
    set_kind("phony")
    on_load(function (target)
        import("foo", {always_build = true})
        import("bar")
        print("foo.add(1,1) = %s", foo.add(1, 1))
        print("foo.sub(1,1) = %s", foo.sub(1, 1))
        print("bar.add(1,1) = %s", bar.add(1, 1))
    end)
```

- `add_moduledirs(dir)` registers a directory as an additional module root. Modules inside are built automatically on first import.
- `{always_build = true}` makes xmake re-check the module sources on every run — essential while iterating. Drop it in production for faster startup.

### When to use which

| Case | Choose |
| --- | --- |
| Need speed + frequent calls | **shared** (no subprocess overhead) |
| Want a trivial implementation, low call count | **binary** (no Lua C API) |
| Need parallel execution across calls | **binary** (each call is its own process) |
| Want to wrap a third-party Lua C module (cjson, etc.) | **shared** |
| Codegen at configure time | Either — binary is simpler |

Reference project in the xmake source: [`tests/projects/other/native_module_cjson`](https://github.com/xmake-io/xmake/tree/master/tests/projects/other/native_module_cjson).

## 10. Common pitfalls

- **Forgetting that description domain is parsed multiple times.** `print` / `os.iorun` / network calls at the top level run on every parse. Move them into `on_load`.
- **Using `require` instead of `import`.** Xmake's module convention is not standard Lua. `require` will *sometimes* work but skips xmake's sandbox, resolution order, and privacy rules. Always `import`.
- **Returning from a module file.** Unnecessary — xmake collects top-level function definitions automatically. A trailing `return M` confuses the loader.
- **Private `_foo` called externally.** `foo._helper(x)` from outside the module fails. Expose a public wrapper or drop the underscore.
- **Native module rebuild not triggering.** Without `{always_build = true}`, xmake only builds the module once. Add it during development.
- **Native module including `lua.h` directly.** Works until you hit a LuaJIT-only build; use `<xmi.h>` for portability.
- **Shelling out with `os.exec("cmd " .. user_input)`.** String-form is shell-parsed. Use `os.execv(program, {args...})` to avoid injection and quoting bugs.

## When to branch out

- Where scripts live vs. description-domain rules → `xmake-style`
- Writing a CLI subcommand (task/plugin) → `xmake-plugins`
- Writing a rule (lifecycle hook tied to a file extension) → `xmake-rules`
- Debugging script behavior (`-vD`, tracebacks, EmmyLua) → `xmake-troubleshooting`, `xmake-dev`
