---
name: xmake-link-order
description: Use when fixing linker errors caused by symbol-order or circular-dependency issues in static libraries — `add_linkorders`, `add_linkgroups` (GNU ld `--start-group`/`--end-group`), whole-archive (`-Wl,--whole-archive`), weak linking, and library strip-off problems. Covers macOS, Linux, MSVC specifics.
---

# Link Order, Link Groups & Whole Archive

When multiple static libraries depend on each other, order matters to the linker. Xmake gives you three knobs to fix order-related link errors without hand-crafting `-Wl,...` flags:

- **`add_linkorders`** — declare that library A must come before library B on the command line.
- **`add_linkgroups`** — wrap a set of libraries in `--start-group/--end-group` (Linux) or the equivalent so the linker resolves circular references.
- **`whole_archive = true`** option on `add_linkgroups` — force-include every object, even ones that look unreferenced (plugin registration, static initializers).

## 1. The underlying problem

A static library is a collection of object files. The GNU linker (and most others) resolves symbols in a single left-to-right pass: when it sees `-lfoo`, it picks up objects that satisfy currently-undefined symbols. Anything not needed *yet* is discarded.

That causes two common bugs:

1. **Wrong order.** `-lbar -lfoo` where `bar` depends on `foo` → undefined symbols from `bar`, because `foo` was consumed before `bar` needed it.
2. **Circular dependency.** `foo` calls into `bar` and `bar` calls into `foo` → impossible with one-pass linking.
3. **Missing static registration.** Plugin/factory patterns where a global constructor registers itself, but no other code directly references it → linker strips the object.

## 2. `add_linkorders` — fix order

```lua
target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_deps("foo", "bar", "baz")
    add_linkorders("bar", "foo")       -- bar depends on foo → bar first, then foo
    add_linkorders("baz", "foo", "bar")
```

`add_linkorders(A, B)` means **A must appear before B** on the linker command line (`-lA ... -lB`). List the dependents *before* their dependencies. Multiple calls accumulate.

Xmake performs a topo-sort over the declared orders and emits the final link line. If you declare a cycle, xmake raises at configure time.

### Order applies to `add_links` too

```lua
target("app")
    add_links("foo", "bar", "baz")
    add_linkorders("bar", "foo")
```

Works for both library dependencies (`add_deps`) and raw links (`add_links` / `add_syslinks`).

## 3. `add_linkgroups` — group libraries for circular deps

```lua
target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_linkgroups("foo", "bar", {group = true})
```

Emits:

```
-Wl,--start-group -lfoo -lbar -Wl,--end-group
```

On Linux/MinGW GNU ld. macOS `ld64` and Windows link.exe handle circular references differently — xmake normalizes the behavior.

Use when:

- `foo.a` and `bar.a` mutually reference each other's symbols.
- You'd otherwise have to list one of them twice (`-lfoo -lbar -lfoo`) — `add_linkgroups` is the clean fix.

## 4. Whole-archive — force-include every object

```lua
target("plugin_host")
    set_kind("binary")
    add_files("src/main.cpp")
    add_linkgroups("plugin_a", "plugin_b", {whole_archive = true})
```

Emits:

```
-Wl,--whole-archive -lplugin_a -lplugin_b -Wl,--no-whole-archive   -- Linux
-Wl,-force_load,<path-to-plugin_a.a>                               -- macOS
/WHOLEARCHIVE:plugin_a.lib                                         -- MSVC
```

Every object in those archives is pulled in — even ones with no outside references. Needed for:

- **Plugin registries** where a global constructor does `register("name", factory)` and there's no other reference.
- **Kernel modules / linker sections** (`__attribute__((constructor))`, `__attribute__((section("...")))`).
- **Language runtimes** that rely on static initializers.

## 5. Combining order + groups + whole archive

```lua
target("app")
    set_kind("binary")
    add_files("src/main.cpp")

    -- regular link deps, in order
    add_deps("core", "utils")
    add_linkorders("core", "utils")

    -- circularly-dependent math libs, grouped
    add_linkgroups("math_core", "math_ext", {group = true})

    -- plugin archives, whole-linked
    add_linkgroups("plugin_json", "plugin_yaml", {whole_archive = true})
```

Xmake arranges these so `core utils` come first (topo order), then the grouped math libs, then the plugin whole-archives. Run `xmake -v` to see the final link command.

## 6. Platform notes

### Linux (GNU ld, lld, mold)

All four features work natively — `--start-group/--end-group`, `--whole-archive/--no-whole-archive`.

### macOS (Apple ld64)

- `--start-group` is a no-op (Apple linker resolves multi-pass by default), so `add_linkgroups(..., {group = true})` is cheap but unnecessary for circular deps on macOS.
- `--whole-archive` maps to `-force_load <path>`. Xmake resolves the archive path automatically when you use `add_linkgroups(..., {whole_archive = true})`.

### Windows (MSVC link.exe)

- Group semantics don't exist; link.exe resolves circular static-lib deps natively.
- Whole-archive maps to `/WHOLEARCHIVE:libname.lib`. Xmake handles it.
- Order is usually not an issue on MSVC; `add_linkorders` still works for cases where it matters.

### Windows (MinGW / gcc)

- Same as Linux GNU ld.

## 7. Diagnosing link errors

```bash
xmake -v                           # show actual link command
xmake -vD 2>&1 | grep -A5 linking
xmake --rebuild                    # force re-link
```

Classic symptoms → fix:

| Symptom | Fix |
| --- | --- |
| `undefined reference to 'bar_func'` and `bar` is linked | `add_linkorders("dependent", "bar")` |
| `undefined reference` in both A and B (both are linked) | `add_linkgroups("A", "B", {group = true})` |
| Plugin registers at runtime but `factory_not_found` | `add_linkgroups("plugin_a", {whole_archive = true})` |
| Works on macOS, fails on Linux | Likely order — Linux GNU ld is stricter |
| Works with `--start-group` manually, clean it up | Use `add_linkgroups(..., {group = true})` |

## 8. Avoiding the problem entirely

- **Merge tightly-coupled libraries into one static lib.** If `foo` and `bar` are mutually dependent, maybe they should be one target.
- **Use shared libraries for plugin systems.** `.so`/`.dll` don't get stripped — `dlopen`/`LoadLibrary` loads them wholesale.
- **Move the registration to the host binary.** Avoids whole-archive entirely by keeping factories in code the host definitely references.
- **Use `set_kind("object")` + `add_files`.** Object libraries sidestep archive strip-off rules: object files are always fully linked.

## 9. Pitfalls

- **`add_linkorders` cycle.** Raises at configure. Check the declared pairs for a cycle.
- **`whole_archive` bloats the binary.** Every object goes in — even test/debug helpers you never wanted. Split the archive before whole-loading.
- **macOS `-force_load` needs absolute paths.** Xmake fills them in; if you build the flags by hand, you must pass the full path.
- **MSVC `/WHOLEARCHIVE` needs the library's base name.** Matching is case-insensitive on Windows, case-sensitive on Linux — stick to lowercase.
- **System libraries.** `add_linkorders` / `add_linkgroups` operate on named libraries (from `add_deps` / `add_links` / `add_syslinks`). For raw `-Wl,...` flags, fall back to `add_ldflags`.

## When to branch out

- Target layout / `add_links` / `add_syslinks` basics → `xmake-targets`
- Cross-compile specific linker flags (`--sdk`/`--cross`) → `xmake-cross-compilation`
- Rules that customize the link step → `xmake-rules`
- Debug "why is this symbol missing" with actual link command → `xmake-troubleshooting`
