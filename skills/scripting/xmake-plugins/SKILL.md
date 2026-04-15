---
name: xmake-plugins
description: Use when invoking built-in Xmake plugins (project generators, compile_commands, doxygen, etc.) or writing a custom task/plugin with `task(...)`.
---

# Xmake Plugins & Tasks

Plugins extend the `xmake` CLI with new subcommands. Xmake ships many built-in plugins; you can also define project-local tasks in `xmake.lua`.

## Common built-in plugins

```bash
xmake project -k compile_commands        # compile_commands.json (clangd/LSP)
xmake project -k cmakelists              # export CMakeLists.txt
xmake project -k ninja                   # build.ninja
xmake project -k vsxmake -m "debug,release"   # Visual Studio
xmake project -k xcode                   # Xcode

xmake doxygen                            # generate API docs via doxygen
xmake lua <script>                       # run a Lua snippet in xmake env
xmake l path/to/script.lua               # shorthand
xmake macro --begin ... --end            # record a macro of commands
xmake repo --add myrepo https://...      # manage xrepo repositories
```

Auto-regenerate `compile_commands.json` on every build:

```lua
add_rules("plugin.compile_commands.autoupdate", {outputdir = "."})
```

## Listing plugins

```bash
xmake show -l plugins
```

## Defining a custom task

A task is a named script reachable via `xmake <taskname>`.

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
        print("hello, %s!", option.get("name"))
    end)
```

```bash
xmake hello --name=ruki
```

Option spec: `{short, long, kind, default, description}` where `kind` is `"k"` (flag), `"kv"` (key-value), or `"vs"` (values).

## Calling Xmake modules from a task

```lua
task("count_files")
    set_menu {usage = "xmake count_files", description = "Count .cpp files"}
    on_run(function ()
        import("core.project.project")
        for _, target in pairs(project.targets()) do
            local n = #target:sourcefiles()
            print("%s: %d files", target:name(), n)
        end
    end)
```

Commonly imported modules: `core.base.option`, `core.project.project`, `core.project.config`, `core.platform.platform`, `lib.detect.find_tool`, `core.tool.toolchain`.

## Tasks vs rules vs plugins

- **Rule**: shapes *how a target is built* (file extensions, per-target hooks).
- **Task**: adds a *new xmake subcommand* the user can invoke.
- **Plugin**: a task packaged for distribution (same `task(...)` API, lives under `~/.xmake/plugins` or a repo).

## When to branch out

- File-extension handling / codegen → `xmake-rules`
- Configure/build/test commands → `xmake-commands`
