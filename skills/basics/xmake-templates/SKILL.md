---
name: xmake-templates
description: Use when creating a new xmake project from a built-in or third-party template (`xmake create -t <template>`), listing available templates, or authoring and distributing custom templates (built-in, user-global, or via xmake-repo). Covers template layout, `${TARGET_NAME}` / `${FAQ}` substitution, and language selection.
---

# Xmake Templates

A template is a directory tree with an `xmake.lua` and some starter source files. `xmake create` copies it into a new project directory, substituting a few placeholders. Templates come from three places, searched in this order:

1. **xmake-repo templates** — third-party, under any registered global repo (`xmake repo --list`), e.g. `packages/templates` from [`xmake-repo`](https://github.com/xmake-io/xmake-repo/tree/master/templates).
2. **User-global templates** — `~/.xmake/templates/`.
3. **Built-in templates** — shipped with xmake under `$(programdir)/repository/templates/`.

## 1. Listing templates

```bash
xmake create --list                # all languages, all sources
xmake create --list -l c++         # only C++ templates
```

Output groups templates by source (repo / global / builtin) and by language, so you can see which names are available for `-t`.

## 2. Creating a project from a template

```bash
xmake create hello                            # default: c++ console
xmake create -l c        hello                # C, default "console" template
xmake create -l c++ -t console    myapp
xmake create -l c++ -t static     mylib
xmake create -l c++ -t shared     mydyn
xmake create -l c   -t module     mymod
xmake create -l rust hello
xmake create -l go   -t console hello
```

Flags:

- `-l, --language` — language (`c`, `c++`, `rust`, `go`, `swift`, `objc`, `cuda`, `dlang`, `fortran`, `kotlin`, `nim`, `pascal`, `vala`, `zig`, `csharp`).
- `-t, --template` — template id inside that language.
- `-P <dir>` — create into a specific directory instead of the current one.
- The positional arg is the target name (defaults to the directory basename or `demo`).

### Third-party templates from xmake-repo

Qt, SDL, Python bindings, Linux drivers, raylib, verilator, wxWidgets, etc. are shipped in `xmake-repo/templates/`. They live under nested template ids using `.` as separator:

```bash
xmake create -l c++ -t qt.widgetapp myqtapp
xmake create -l c++ -t qt.quickapp  myquick
xmake create -l c++ -t wxwidgets    mywx
xmake create -l c++ -t python.pybind11 mybind
xmake create -l c   -t sdl          mysdl
xmake create -l c   -t raylib       mygame
xmake create -l c   -t linux.driver mydriver
```

`qt.widgetapp` resolves to `<repo>/templates/c++/qt/widgetapp/`. Each path segment between dots is a directory level.

### Targeting a specific directory

```bash
xmake create -P myproj -l c++ -t console hello
cd myproj && xmake
```

## 3. What a template actually is

A template is just:

```
<root>/<language>/<template-id-path>/
├── xmake.lua                # with ${TARGET_NAME} / ${FAQ} placeholders
└── src/
    └── main.cpp             # or whatever source files the template needs
```

When you run `xmake create`, xmake copies the entire directory tree to the destination and substitutes placeholders in both filenames and file contents.

### Built-in placeholders

Verified from `xmake/actions/create/template.lua`:

| Placeholder | Value |
| --- | --- |
| `${TARGET_NAME}` | The positional project/target name passed to `xmake create` |
| `${FAQ}` | Contents of `$(programdir)/scripts/faq.lua` — a short FAQ block appended to the generated `xmake.lua` |

Example built-in `c/console/xmake.lua`:

```lua
add_rules("mode.debug", "mode.release")

target("${TARGET_NAME}")
    set_kind("binary")
    add_files("src/*.c")

${FAQ}
```

After `xmake create -l c hello`, the emitted `xmake.lua` has `target("hello")` and the FAQ block inlined.

## 4. Authoring a custom template

### Step 1 — pick a location

Either of these is picked up automatically; no registration step needed:

- **User-global** — `~/.xmake/templates/<language>/<template-id>/` — personal, visible on your machine only.
- **Inside an xmake-repo clone** — `<repo>/templates/<language>/<template-id>/` — distributable to anyone who adds your repo.

Template IDs can be nested: `~/.xmake/templates/c++/game/sdl2/` → `-t game.sdl2`.

### Step 2 — create the tree

```
~/.xmake/templates/c++/game/sdl2/
├── xmake.lua
├── src/
│   └── main.cpp
└── assets/
    └── .gitkeep
```

Template `xmake.lua`:

```lua
add_rules("mode.debug", "mode.release")
set_languages("c++17")

add_requires("libsdl2")

target("${TARGET_NAME}")
    set_kind("binary")
    add_files("src/*.cpp")
    add_packages("libsdl2")

${FAQ}
```

Template `src/main.cpp`:

```cpp
#include <SDL2/SDL.h>
int main(int, char**) {
    SDL_Init(SDL_INIT_VIDEO);
    SDL_Quit();
    return 0;
}
```

### Step 3 — verify

```bash
xmake create --list -l c++         # your template should appear under "c++/game.sdl2"
xmake create -l c++ -t game.sdl2 myshooter
cd myshooter
xmake
```

### Placeholder substitution scope

- Applies to **file contents** of every copied file.
- Applies to **filenames** as well — a file named `${TARGET_NAME}.h` becomes `myshooter.h`.
- Only the two placeholders above are substituted. Do **not** invent new `${...}` — they will be left as literal text in the output.

### Multi-target / multi-file templates

Nothing special — just drop more files into the template tree. Globs in the template `xmake.lua` (`add_files("src/*.cpp")`) will pick up whatever the user adds later, so keep file lists glob-based rather than enumerated.

## 5. Distributing a template via xmake-repo

If you want other people to use your template:

1. Fork [`xmake-repo`](https://github.com/xmake-io/xmake-repo).
2. Drop your template under `templates/<language>/<id...>/`.
3. Submit a PR.

Or, for a private template library, host your own repo:

```bash
xrepo add-repo my-templates https://github.com/me/my-xmake-templates
```

Once the repo is registered globally, its `templates/` directory is picked up by `xmake create --list` and usable via `-t`.

## 6. Common pitfalls

- **Template id path separators.** Use `.` on the CLI (`-t qt.widgetapp`) — not `/`. xmake splits on `.` to walk the template tree.
- **`--list` doesn't show my template.** Either the directory is not at `<root>/<language>/<id>/`, or there is no `xmake.lua` in it. `templatedir()` requires both.
- **Placeholder not substituted.** Only `${TARGET_NAME}` and `${FAQ}` are real. Anything else you write with `${...}` is left verbatim.
- **Wrong language flag.** `-l` must exactly match the directory name under `templates/`. `c++` not `cpp`; `objc++` not `objcxx`.
- **Path traversal protection.** `xmake create` rejects `.`, `..`, `/`, `\`, `:`, or NUL in template ids and language names. Keep template ids to plain identifiers separated by dots.
- **Using `xmake create` for scaffolding an *existing* project.** It creates files in an otherwise empty directory; running it inside a populated directory will error out if files would be overwritten. Use a fresh directory or `-P <new-dir>`.

## When to branch out

- Basic `xmake create` flow as part of "getting started" → `xmake-basics`
- Adding packages the template depends on → `xmake-packages`
- Writing the `xmake.lua` body that ends up in the template → `xmake-targets`, `xmake-rules`
