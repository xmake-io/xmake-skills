---
name: xmake-project-generator
description: Use when exporting an xmake project to another build system or IDE — `compile_commands.json` for clangd/LSP, `CMakeLists.txt`, `build.ninja`, Visual Studio solution (`vsxmake`), or an Xcode project. Covers `xmake project -k <kind>` and how each generator is typically used.
---

# Xmake Project Generators

`xmake project` exports your xmake project into another build system's format. The core command is:

```bash
xmake project -k <kind> [options]
```

Use a generator when you need to hand the project to a tool that does not speak xmake: an IDE, an LSP, a CI system that only runs CMake, or someone reviewing the build graph.

## Available kinds

```bash
xmake project -k compile_commands    # compile_commands.json (clangd/ccls/LSP)
xmake project -k cmakelists          # CMakeLists.txt
xmake project -k ninja               # build.ninja
xmake project -k vsxmake             # Visual Studio solution (xmake-driven)
xmake project -k vs                  # Visual Studio solution (native, older)
xmake project -k xcode               # Xcode .xcodeproj
xmake project -k makefile            # GNU Makefile
```

List available kinds on your install:

```bash
xmake project --help
```

## 1. `compile_commands.json` (clangd / ccls / LSP)

This is by far the most-used generator. It emits a `compile_commands.json` that clangd / ccls / nvim-lsp / VS Code's `clangd` extension read to provide cross-project navigation and diagnostics.

### One-shot generation

```bash
xmake project -k compile_commands
xmake project -k compile_commands -o .              # write to current dir
xmake project -k compile_commands build              # write to build/
```

The file is written to the project root by default. Pass an output dir as positional argument.

### Auto-regenerate on every build (recommended)

Add this to `xmake.lua` once:

```lua
add_rules("plugin.compile_commands.autoupdate", {outputdir = "."})
```

Now `compile_commands.json` is refreshed on every `xmake build`, including when you add files or change flags. No more stale LSP data.

Options:

- `outputdir = "."` — where to write the file (use `.` so clangd finds it without extra config).
- `lsp = "clangd"` — tweak for non-clangd LSPs (rarely needed).

### Per-mode/arch captures

```bash
xmake f -m debug   && xmake project -k compile_commands -o build/debug
xmake f -m release && xmake project -k compile_commands -o build/release
```

Useful if your LSP config points at a specific config.

## 2. `CMakeLists.txt`

Exports a CMake project equivalent. Handy when a downstream consumer can only build with CMake, or when migrating away from xmake.

```bash
xmake project -k cmakelists
xmake project -k cmakelists -o build/cmake
```

Caveats:

- It's a **one-way export** — regenerate from xmake, don't edit the generated CMakeLists by hand.
- Complex features (custom rules with `on_buildcmd_file`, codegen, xmake-specific toolchains) may not translate 1:1. Review the output.
- Re-run after changing `xmake.lua`.

## 3. `build.ninja`

Skip xmake's runner and drive the build with `ninja` directly. Occasionally useful for integration with tools that already sit on top of ninja.

```bash
xmake project -k ninja
ninja -C build
```

You still need xmake to configure (`xmake f ...`) first — ninja just executes the graph.

## 4. Visual Studio — `vsxmake` vs `vs`

### `vsxmake` (preferred)

Generates a `.sln` + project files that **shell out to xmake** for the actual build. Keeps xmake as the source of truth: any edit to `xmake.lua` is reflected immediately in VS without regenerating.

```bash
xmake project -k vsxmake                                    # default modes+archs
xmake project -k vsxmake -m "debug,release"                 # specific modes
xmake project -k vsxmake -m "debug,release" -a "x64,x86"    # modes + archs
```

Opens cleanly in Visual Studio 2019/2022. Debugging, IntelliSense, and breakpoints work.

### `vs` (native)

Generates a native VS project that uses MSBuild end-to-end — no xmake at build time. Less fidelity; use only if you need to hand the solution to a team that never wants to install xmake.

```bash
xmake project -k vs -m "debug,release"
```

## 5. Xcode

```bash
xmake project -k xcode
xmake project -k xcode -m "debug,release"
```

Generates an `.xcodeproj`. Use for iOS/macOS debugging, Instruments profiling, or distributing an Xcode project to Apple-ecosystem teammates.

Caveats similar to `vs`: regenerate from xmake when the build changes; don't edit the Xcode project directly.

## 6. GNU Makefile

```bash
xmake project -k makefile
make
```

Rarely needed, but useful for embedded CI that only has `make`.

## Common flags

| Flag | Purpose |
| --- | --- |
| `-k, --kind=<kind>` | Which generator |
| `-m, --modes="debug,release"` | Emit multiple build modes (vs/vsxmake/xcode) |
| `-a, --archs="x64,x86"` | Emit multiple architectures |
| `-o, --outputdir=<dir>` | Where to write the generated files |
| `[positional]` | Same as `-o` — target directory |

## Workflow recipes

### LSP setup (one-time)

```lua
-- xmake.lua
add_rules("plugin.compile_commands.autoupdate", {outputdir = "."})
```

Then:

```bash
xmake f -m debug
xmake                       # build + auto-generate compile_commands.json
```

Point clangd at the project root and you're done.

### Hand a CMake tarball to a client

```bash
xmake project -k cmakelists -o dist/cmake
cp -r src include dist/cmake/
tar czf myproject-cmake.tar.gz -C dist cmake
```

### Open in Visual Studio with multiple configs

```bash
xmake project -k vsxmake -m "debug,release" -a "x64,x86"
start vsxmake/vs2022/myproject.sln
```

### Drive ninja from CI

```bash
xmake f -m release
xmake project -k ninja
ninja -C build -j 8
```

## Pitfalls

- **Forgetting to run `xmake f` first.** Generators read the configured state (plat/arch/mode). Run `xmake f -m release` (or whatever) before generating, otherwise you export whatever happens to be in `.xmake/`.
- **Editing generated files.** Every generator is one-way. Changes to `CMakeLists.txt`, `.vcxproj`, `.xcodeproj` are lost on the next regeneration. Treat them like compiler output.
- **Stale `compile_commands.json`.** If your LSP is showing errors for files that xmake builds fine, regenerate — or use `plugin.compile_commands.autoupdate` so it happens automatically.
- **`vs` vs `vsxmake`.** If you hit weird incremental-build bugs or missing custom rules in Visual Studio, you probably want `vsxmake` instead of `vs`. `vsxmake` keeps xmake in the loop.
- **Generated CMake doesn't include custom rules.** Anything that relies on xmake's `rule()` with C/Lua-side logic (codegen, file-extension handlers) may not survive the export. Verify the output.

## When to branch out

- Configuring the underlying project → `xmake-targets`, `xmake-rules`
- Building / running / packaging without a generator → `xmake-commands`
- IDE integration plugins in general → `xmake-plugins`
