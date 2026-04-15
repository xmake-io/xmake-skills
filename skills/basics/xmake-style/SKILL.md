---
name: xmake-style
description: Use when writing or reviewing an `xmake.lua` (project, package recipe, or toolchain) and you want to follow idiomatic xmake style — description vs script domain separation, naming, indentation, `set_` vs `add_`, option/config organization, and common conventions used across xmake-repo and the xmake project itself. Apply this alongside `xmake-targets`/`xmake-packages`/etc. as a stylistic overlay.
---

# Xmake Configuration Style & Conventions

`xmake.lua` is Lua, but idiomatic xmake code looks less like Lua and more like a declarative configuration. Following the conventions below keeps files readable, fast to parse, and consistent with the style used throughout [`xmake`](https://github.com/xmake-io/xmake) and [`xmake-repo`](https://github.com/xmake-io/xmake-repo).

## 1. The description / script split (the single most important rule)

`xmake.lua` has **two domains** and they have very different rules.

### Description domain — keep it declarative

This is the top-level body of `target()`, `option()`, `package()`, `task()`, `rule()`, and their `set_xxx`/`add_xxx` calls.

```lua
target("app")
    set_kind("binary")
    add_files("src/*.cpp")
    add_defines("DEBUG")
    add_syslinks("pthread")
```

Rules for the description domain:

- **Treat it as config, not code.** Basic `if is_plat(...)` and `for _, x in ipairs({...})` are fine; complex logic is not.
- **It is parsed multiple times.** Every `set_`/`add_` call may run more than once as xmake re-enters the file at different configuration stages. **Never `print()` here** (you will see it twice), and never do expensive work (I/O, git, network, shell).
- **Many APIs are forbidden or read-only here.** `os.getenv` is read-only; most mutating `os.*` calls are blocked.
- If logic is not a trivial `if` or `for`, move it to the script domain.

### Script domain — put complex logic here

Anything inside `on_load`, `on_config`, `before_build`, `after_build`, `on_install`, `on_test`, etc., is the script domain. It runs **once per lifecycle hook** and has the full xmake Lua environment.

```lua
target("app")
    set_kind("binary")
    add_files("src/*.cpp")
    on_load(function (target)
        if is_plat("linux", "macosx") then
            target:add("links", "pthread", "m", "dl")
        end
    end)
    after_build(function (target)
        import("core.project.config")
        os.cp(target:targetfile(), path.join(config.buildir(), "dist"))
    end)
```

Use `on_load` for dynamic per-target configuration that would be ugly as nested `if` blocks at description level.

### Extract long scripts to separate files

Once a hook body exceeds ~15 lines, move it out:

```lua
target("app")
    on_load("modules.app.load")
    on_install("modules.app.install")
```

With files at `modules/app/load.lua` and `modules/app/install.lua`, each exporting a `main` function. Keeps the main `xmake.lua` scannable.

## 2. Indentation and layout

The conventions used throughout the xmake project itself:

```lua
-- 4 spaces, never tabs
add_rules("mode.debug", "mode.release")

-- blank line between top-level blocks
add_requires("fmt 10.x", "spdlog")

-- everything inside a target() is indented one level
target("mylib")
    set_kind("static")
    add_files("src/lib/*.cpp")
    add_includedirs("include", {public = true})
    add_packages("fmt")

target("app")
    set_kind("binary")
    add_files("src/app/*.cpp")
    add_deps("mylib")
```

- **4 spaces per level.** No tabs. The xmake CONTRIBUTING guide makes this explicit.
- **No explicit `target_end()`** unless you have to. A new top-level call (`target`, `option`, `package`, `rule`, `task`) implicitly closes the previous one.
- **Blank line between blocks.** Makes visual scanning easier when a file has many targets.
- **Related `add_*` calls stay grouped.** Put `add_files` / `add_headerfiles` together; put `add_includedirs` / `add_defines` together; put `add_packages` / `add_deps` together.
- **Comments use `--`.** Use `--` for single-line comments. Avoid `--[[ ]]` blocks unless you are commenting out several lines temporarily.

### Parentheses are optional for single-arg string calls

Both of these are valid:

```lua
target("test")
    set_kind("binary")
    add_files("src/*.c")
```

```lua
target "test"
    set_kind "binary"
    add_files "src/*.c"
```

**Pick one and stick with it per file.** The xmake and xmake-repo codebases predominantly use parentheses — match that style unless you have a reason.

## 3. Naming conventions

- **Target names**: lowercase, `-` or `_` for word separators. `mylib`, `my-lib`, `my_lib`. Match the produced binary name when it makes sense.
- **Options**: same — lowercase with `_`. Prefix with `enable_` / `with_` / `has_` to signal intent (`enable_foo`, `with_openssl`, `has_avx2`).
- **Rules**: dotted namespaces for grouping (`mycompany.codegen`, `mycompany.protobuf`). Match how built-in rules are named (`mode.debug`, `c++.unity_build`, `plugin.compile_commands.autoupdate`).
- **Custom toolchains**: short, lowercase, no prefix. `myclang`, `arm-muslgcc`.
- **Packages** (in `xmake-repo` recipes): exactly the upstream name, lowercase, `-`-separated. Match what `add_requires` users will type.

## 4. `set_` vs `add_`: pick the right one

- **`set_xxx`** — replaces. Use for **single-valued** properties: `set_kind`, `set_version`, `set_languages`, `set_optimize`, `set_symbols`, `set_default`, `set_toolchains`.
- **`add_xxx`** — appends. Use for **list-valued** properties: `add_files`, `add_includedirs`, `add_defines`, `add_links`, `add_deps`, `add_packages`, `add_cxflags`.

If you call `set_xxx` twice for the same key, the second wins. If that is what you want, fine — but it is almost always a mistake.

```lua
-- ✗ wrong: second set_languages replaces the first
set_languages("c++17")
set_languages("c++20")

-- ✓ just set it once, at the level you want
set_languages("c++20")
```

## 5. Scope & visibility (`{public = true}`)

Use `{public = true}` for **anything a dependent target needs to see**: public headers, public defines, public link libraries.

```lua
target("mylib")
    set_kind("static")
    add_files("src/*.cpp")
    add_includedirs("include", {public = true})    -- dependents get -Iinclude
    add_defines("MYLIB_STATIC", {public = true})   -- and this define
```

- Private by default is correct: a dependent target should not inherit your `src/` include dir or your internal defines.
- `{force = true}` bypasses xmake's automatic flag-detection filter. Use sparingly and only when you know the flag is supported.

## 6. Conditionals: prefer `is_plat` / `is_arch` / `is_mode`

Idiomatic:

```lua
if is_plat("windows") then
    add_defines("WIN32_LEAN_AND_MEAN")
elseif is_plat("linux", "macosx") then
    add_syslinks("pthread")
end

if is_mode("debug") then
    add_defines("DEBUG")
    set_symbols("debug")
end
```

Avoid reaching into `os.host()` / `os.arch()` at description level for platform gating — that is the *host*, not the *target*. `is_plat`/`is_arch` reflect the configured target and are the right answer 99% of the time.

## 7. Options — declaration style

```lua
option("enable_foo")
    set_default(false)
    set_showmenu(true)
    set_description("Enable the foo subsystem")
    set_category("feature")         -- optional grouping in --menu
option_end()
```

Conventions:

- Always `set_showmenu(true)` if the option is meant for end users — otherwise it's invisible to `xmake f --menu`.
- Always `set_description(...)` — the description is what shows up in the configure menu.
- Use `set_default(...)` to pin a default; do not rely on `nil`.
- Use `set_values(...)` to constrain to an enum when applicable.
- For build-flag-style options, prefer `has_config("name")` over `get_config("name")` in the target body unless you need the actual value.

## 8. Packages — declaration style

In a project `xmake.lua`:

```lua
-- all add_requires at the top of the file
add_requires("fmt 10.x")
add_requires("spdlog", {configs = {header_only = false}})
add_requires("openssl", {system = false})

target("app")
    add_packages("fmt", "spdlog", "openssl")
```

- Declare `add_requires` **near the top**, before any `target()`. Keeps the dependency surface visible in one place.
- Pin major versions for anything you actually care about stability for (`"fmt 10.x"`, `"boost 1.84.x"`).
- Prefer `add_packages(...)` over manually setting `add_links`/`add_includedirs` — the package integration does the right thing automatically.

In a **package recipe** (`packages/<a>/<name>/xmake.lua` in xmake-repo):

```lua
package("mylib")
    set_homepage("https://example.com/mylib")
    set_description("A short one-line description")
    set_license("MIT")

    add_urls("https://github.com/example/mylib/archive/v$(version).tar.gz",
             "https://github.com/example/mylib.git")

    add_versions("1.2.3", "abcdef...sha256...")

    add_configs("shared", {description = "Build shared library", default = false, type = "boolean"})
    add_configs("with_ssl", {description = "Enable SSL support", default = true, type = "boolean"})

    add_deps("cmake")
    if is_plat("linux") then
        add_syslinks("pthread")
    end

    on_install(function (package)
        local configs = {"-DBUILD_TESTS=OFF"}
        table.insert(configs, "-DBUILD_SHARED_LIBS=" .. (package:config("shared") and "ON" or "OFF"))
        import("package.tools.cmake").install(package, configs)
    end)

    on_test(function (package)
        assert(package:has_cfuncs("mylib_init", {includes = "mylib.h"}))
    end)
package_end()
```

Recipe conventions:

- `set_homepage` / `set_description` / `set_license` in that order, at the top.
- Multiple `add_urls` for mirror fallback; the `.git` URL last.
- `add_configs` names **lowercased with underscores**: `shared`, `with_ssl`, `header_only`, `enable_foo`. Match what users already type in other recipes.
- `add_versions` kept in descending order (newest at top) in xmake-repo.
- `on_install` uses `import("package.tools.cmake")` (or `autoconf`, `meson`, `make`, `msbuild`, `xmake`) — almost always one of these. Don't hand-roll.
- `on_test` asserts a real symbol with `has_cfuncs` / `has_cxxtypes` / `check_csnippets`. Empty `on_test` is a red flag.

## 9. Don't reinvent common patterns

- **Build modes** → `add_rules("mode.debug", "mode.release")`. Don't hand-roll debug/release defines.
- **`compile_commands.json`** → `add_rules("plugin.compile_commands.autoupdate", {outputdir = "."})`.
- **Unity builds** → `add_rules("c++.unity_build")`, not hand-crafted aggregating files.
- **Qt apps** → `add_rules("qt.widgetapp" / "qt.quickapp")`, not manual `uic`/`moc`/`rcc` invocations.
- **Generating a config header** → `add_configfiles("config.h.in")` + `set_configvar`, not a `before_build` Lua shell script.

If the built-in exists, use it. It handles edge cases (cross-compilation, multiple platforms, caching) that your hand-rolled version will not.

## 10. Things to avoid

| Anti-pattern | Why | Do this instead |
| --- | --- | --- |
| `print(...)` at description level | Runs multiple times | Use `cprint` in `on_load` / `on_config`, or `utils.vprint` gated by `-v` |
| Complex `for` / nested `if` at description level | Parsed multiple times; harder to reason about | Move into `on_load(function(target) ... end)` |
| `os.iorun` / git calls at description level | Runs during every parse | Move into `on_load` or a task |
| `add_cxflags("-std=c++20", {force = true})` | Bypasses xmake's language handling | `set_languages("c++20")` |
| `add_links("pthread")` on cross-platform code | Not a library on Windows | `add_syslinks("pthread")` |
| `set_languages` called per-target when every target uses the same standard | Duplication | Call once at the top of `xmake.lua` |
| Using absolute paths | Not portable | Paths relative to `xmake.lua`; xmake resolves them |
| Re-exporting every header with `add_includedirs` | Pollutes dependents | `{public = true}` only on the public include dir |
| Empty `on_test` in a package recipe | Passes trivially | Use `has_cfuncs` / `check_csnippets` to actually exercise the library |

## 11. Commit-message convention (contributor-facing)

From `CONTRIBUTING.md` of both `xmake` and `xmake-repo`:

- Commit messages in **English**.
- Use descriptive summaries: `add ...`, `fix ...`, `improve ...`, `update ...`.
- New or changed public API should go through an issue/RFC first rather than straight into a PR.
- Indent edited code with 4 spaces, never tabs — same as the `xmake.lua` rule.

## When to branch out

- The actual semantics of `target` / `add_files` / `add_deps` → `xmake-targets`
- The actual semantics of `option` / `has_config` → `xmake-options`
- The actual semantics of `add_requires` / `add_packages` → `xmake-packages`
- Recipe authoring workflow and testing → `xmake-repo-testing`
- Rule/toolchain semantics → `xmake-rules` / `xmake-toolchains`
