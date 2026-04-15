---
name: xmake-custom-plugins
description: Use when authoring a custom xmake plugin / task — defining `task(...)` with `set_menu`, running logic in `on_run`, packaging as a standalone plugin under `~/.xmake/plugins/` or inside an xmake-repo, using `import()` to reuse xmake core modules, and distributing plugins. Complements the lighter `xmake-plugins` skill which covers *using* plugins.
---

# Authoring Custom Plugins & Tasks

A **task** is a named function reachable via `xmake <taskname>` on the CLI. A **plugin** is just a task packaged for reuse / distribution. Same API — the only difference is where the code lives.

This skill focuses on **writing** plugins. For using built-in plugins (`xmake project -k compile_commands`, etc.), see `xmake-plugins`.

## 1. Minimal task in `xmake.lua`

```lua
task("hello")
    set_menu {
        usage       = "xmake hello [options]",
        description = "Print a greeting",
        options     = {
            {"n", "name", "kv", "world", "Name to greet"},
            {nil, "loud", "k",  nil,     "Shout it"}
        }
    }
    on_run(function ()
        import("core.base.option")
        local name = option.get("name")
        if option.get("loud") then
            cprint("${bright red}HELLO, %s!", name:upper())
        else
            cprint("${green}hello, %s", name)
        end
    end)
```

```bash
xmake hello                  # hello, world
xmake hello --name=ruki      # hello, ruki
xmake hello -n ruki --loud   # HELLO, RUKI!
xmake hello --help           # menu auto-generated
```

## 2. `set_menu` schema

```lua
set_menu {
    usage       = "xmake <task> [options]",     -- one-line usage string
    description = "Short description",          -- listed in `xmake --help`
    options     = {
        -- { short, long, kind, default, description [, extra] }
        {"n", "name",    "kv", "world", "Name to greet"},
        {"v", "verbose", "k",  nil,     "Verbose output"},
        {nil, "files",   "vs", nil,     "Input files"},
        {},                                      -- separator in help output
        {nil, "mode",    "kv", "release", "Build mode",
                          {"debug", "release"}}, -- restricted values
    }
}
```

Option kinds:

| Kind | Meaning | Usage |
| --- | --- | --- |
| `"k"` | Flag (boolean) | `--name`, `-v` |
| `"kv"` | Key-value | `--name=value`, `-n value` |
| `"vs"` | Values list | `--files=a.cpp --files=b.cpp` or positional |

Short name is optional (`nil` for long-only).

## 3. `on_run` body

`on_run` is the task entry point. It has full script-domain access — import any core module, read project state, shell out, etc.

```lua
on_run(function ()
    import("core.base.option")
    import("core.project.project")
    import("core.project.config")

    -- read options
    local name   = option.get("name")
    local loud   = option.get("loud")
    local files  = option.get("files")            -- list for "vs" kind

    -- inspect the project
    for target_name, target in pairs(project.targets()) do
        print("%s -> %s", target_name, target:targetfile())
    end

    -- current config
    print("plat=%s arch=%s mode=%s", config.plat(), config.arch(), config.mode())
end)
```

## 4. Splitting the body into modules

Once `on_run` grows, move it:

```
myproject/
  xmake.lua
  plugins/
    hello/
      xmake.lua      # task definition
      main.lua       # entry
      utils.lua      # helpers
```

```lua
-- plugins/hello/xmake.lua
task("hello")
    set_menu { ... }
    on_run("main")       -- "main" = main.lua in the same directory
```

```lua
-- plugins/hello/main.lua
import("core.base.option")
import(".utils")        -- . = current dir

function main()
    local name = option.get("name")
    utils.greet(name)
end
```

```lua
-- plugins/hello/utils.lua
function greet(name)
    cprint("${bright green}hi, %s${clear}", name)
end
```

Include the plugin from the project:

```lua
-- main xmake.lua
includes("plugins/hello")
```

Now `xmake hello` works.

## 5. Packaging as a standalone plugin

A plugin is just a task directory installed somewhere xmake searches:

1. `~/.xmake/plugins/<name>/` — per-user plugins
2. Bundled in an xmake repository under `plugins/` — distributable via `xrepo add-repo`
3. `$(programdir)/plugins/` — built-in plugins (read-only)

Minimum layout:

```
~/.xmake/plugins/hello/
├── xmake.lua           # task(...) definition
└── main.lua            # implementation
```

After dropping the files, `xmake hello` works **in any project** — no `includes(...)` needed.

## 6. Using xmake core modules from a plugin

Frequently imported:

```lua
import("core.base.option")                    -- task options
import("core.base.task")                      -- task.run("other-task")
import("core.base.global")                    -- global config
import("core.project.project")                -- project:targets(), project:rootfile(), etc.
import("core.project.config")                 -- plat/arch/mode
import("core.platform.platform")              -- platform info
import("core.tool.toolchain")                 -- resolved toolchain
import("core.tool.compiler")                  -- invoke a compiler
import("core.tool.linker")                    -- invoke a linker
import("core.package.package")                -- installed package info
import("lib.detect.find_tool")                -- probe PATH
import("lib.detect.find_package")             -- probe packages
import("net.http")                            -- HTTP client
import("devel.git")                           -- git clone/ls-remote
import("utils.archive")                       -- zip/tar extract
```

## 7. Calling one task from another

```lua
on_run(function ()
    import("core.base.task")
    task.run("build", {target = "app"})    -- run xmake build app
    task.run("custom-task")
end)
```

## 8. Plugin that operates on the current project

```lua
task("stats")
    set_menu {
        usage = "xmake stats",
        description = "Show file counts per target"
    }
    on_run(function ()
        import("core.project.project")
        for name, target in pairs(project.targets()) do
            cprint("${bright}%s${clear}: %d files", name, #target:sourcefiles())
        end
    end)
```

## 9. Plugin that takes positional arguments via `"vs"`

```lua
task("sha256")
    set_menu {
        usage = "xmake sha256 <file>...",
        description = "Hash one or more files",
        options = {
            {nil, "files", "vs", nil, "Files to hash"}
        }
    }
    on_run(function ()
        import("core.base.option")
        for _, f in ipairs(option.get("files") or {}) do
            print("%s  %s", hash.sha256(f), f)
        end
    end)
```

```bash
xmake sha256 a.tar b.tar c.tar
```

## 10. Plugin that uses a native module

When the plugin does heavy work in C/C++ (hash, parse, math), pair it with a native module — see `xmake-scripting`. Short version:

```lua
-- plugins/hello/xmake.lua
includes("modules")              -- builds a native .so/.dll module
task("hello")
    on_run(function ()
        import("hello_native")
        print(hello_native.compute(42))
    end)
```

## 11. Distributing a plugin

### Via an xmake repository

1. Put the plugin in `my-repo/plugins/hello/`.
2. `xrepo add-repo my-repo https://github.com/me/my-repo.git`
3. Xmake automatically picks up plugins under any registered repo.

### Via `~/.xmake/plugins/`

Just `git clone https://github.com/me/hello-plugin ~/.xmake/plugins/hello`. Done.

### As part of a project

`includes("plugins/*")` in the project's top-level `xmake.lua` — picks up every task inside `plugins/`.

## 12. Debugging plugins

```bash
xmake hello -vD                           # verbose + Lua tracebacks
XMAKE_PROFILE=stuck xmake hello           # if it hangs
xmake l plugins/hello/main.lua            # run the file directly in xmake Lua env
```

Set `print(...)` liberally — `on_run` runs in the script domain, so all logging primitives are available.

## Pitfalls

- **Using `set_menu { ... }` vs `set_menu({...})`.** Both work (Lua sugar for a single table arg). Inside `options`, use explicit braces.
- **Forgetting `on_run`.** A task without `on_run` is just a menu entry — `xmake hello` prints help and exits.
- **Using description-domain calls inside the task.** `task()` is at top level; everything mutating belongs in `on_run`. `cprint` at top level runs on every parse.
- **Option kind mismatch.** `"k"` flags are booleans — `option.get("flag")` returns `true`/`false`, not a value. `"kv"` returns the string value.
- **Task name collision.** If your plugin's name matches a built-in (`build`, `run`, `install`), xmake raises. Pick a unique prefix (`mycompany.build`).
- **Side effects at plugin load time.** `includes()` parses the plugin's `xmake.lua`, so don't put `os.exec` at top level — it runs even if the user never invokes the task.

## When to branch out

- Using built-in plugins (project generator, compile_commands, etc.) → `xmake-plugins`
- Writing a rule (file-extension handler, not a CLI subcommand) → `xmake-rules`
- Scripting and `import()` basics → `xmake-scripting`
- Common script modules (`os`/`io`/`path`) → `xmake-script-modules`
- Native (C/C++) modules used by plugins → `xmake-scripting`
