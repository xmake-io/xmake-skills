---
name: xmake-xpack
description: Use when producing distributable packages from an xmake project — zip/tar.gz/deb/rpm/nsis/wix/srpm/pacman/runself/zst etc. — via `xpack(...)` in `xmake.lua` and the `xmake pack` CLI. Covers package metadata, format selection, install layout, components, custom install commands, and CLI flags.
---

# XPack: Package Configuration & CLI

`xpack` is xmake's built-in packaging plugin. Declare a package in `xmake.lua`, then run `xmake pack` to produce distributable archives or installers. Unlike `xmake install` (which installs to a prefix), `xpack` emits a file you can ship.

Supported formats: `zip`, `targz`, `tar.gz`, `srczip`, `srctargz`, `nsis` (Win installer), `wix` (MSI), `deb`, `rpm`, `srpm`, `pacman`, `runself` (self-extracting), `zst`.

## 1. Minimal example

```lua
-- xmake.lua
includes("@builtin/xpack")

target("myapp")
    set_kind("binary")
    set_version("1.0.0")
    add_files("src/*.cpp")

xpack("myapp")
    set_formats("zip", "targz", "nsis", "deb")
    set_homepage("https://example.com")
    set_license("Apache-2.0")
    set_description("A short one-line description")
    add_targets("myapp")
```

Build the package:

```bash
xmake f -m release
xmake
xmake pack
```

Output lands under `build/xpack/<name>/`.

`includes("@builtin/xpack")` is required — xpack is a plugin, not a built-in API.

## 2. Metadata (all optional but recommended)

```lua
xpack("myapp")
    set_version("1.0.0")                            -- or inherits from add_targets
    set_title("My Application")                     -- short UI label
    set_description("Detailed one-paragraph summary.")
    set_homepage("https://example.com")
    set_author("Ruki <ruki@example.com>")
    set_maintainer("Ops Team <ops@example.com>")
    set_copyright("Copyright (c) 2025, Example Corp")
    set_license("Apache-2.0")                       -- SPDX identifier
    set_licensefile("LICENSE.md")                   -- include the license text
    set_company("Example Corp")
```

Version lookup order: `xpack:set_version` > inherited from first `add_targets` > project-level `set_version`.

## 3. Format selection

```lua
set_formats("zip", "targz")                         -- generic archives
set_formats("nsis")                                 -- Windows installer
set_formats("wix")                                  -- Windows MSI
set_formats("deb", "rpm")                           -- Linux packages
set_formats("pacman")                               -- Arch
set_formats("runself")                              -- self-extracting .run
set_formats("srczip", "srctargz")                   -- source-only
```

Or pick the format at `xmake pack` time:

```bash
xmake pack -f zip
xmake pack -f nsis
xmake pack -f deb myapp
xmake pack -f "zip,targz,deb" myapp                 # multiple at once
```

The CLI `-f` overrides what `set_formats` declares.

## 4. Install layout

```lua
xpack("myapp")
    add_targets("myapp")                            -- main binaries/libs
    set_prefixdir("myapp-$(version)")               -- top-level dir inside archive
    set_bindir("bin")                               -- where binaries go
    set_libdir("lib")                               -- where libraries go
    set_includedir("include")                       -- where headers go
```

Add extra content:

```lua
    add_sourcefiles("README.md", "docs/*.md")       -- include in source pkgs
    add_installfiles("assets/icon.png", {prefixdir = "share/myapp"})
    add_installfiles("config/*.json", {prefixdir = "etc/myapp"})
```

`add_installfiles` is the general-purpose "ship this file at this path" — use it for anything that isn't a target output.

## 5. Components (optional install groups)

Split a package into optional components users can pick:

```lua
xpack("myapp")
    set_formats("nsis")
    add_components("app", "docs", "examples")

xpack_component("app")
    set_title("Main application")
    set_default(true)
    add_targets("myapp")

xpack_component("docs")
    set_title("Documentation")
    set_default(false)
    add_installfiles("docs/**", {prefixdir = "share/doc"})

xpack_component("examples")
    set_title("Example code")
    set_default(false)
    add_installfiles("examples/**", {prefixdir = "share/myapp/examples"})
```

Components map to NSIS/WiX/deb-dpkg component selectors in the installer UI.

## 6. Custom install/uninstall commands

For installers that need to run scripts (register services, update PATH, create symlinks):

```lua
xpack("myapp")
    on_installcmd(function (package, batchcmds)
        batchcmds:mkdir(package:installdir("etc"))
        batchcmds:cp("config/*.json", package:installdir("etc"))
        if is_plat("linux") then
            batchcmds:runv("systemctl", {"daemon-reload"})
        end
    end)

    on_uninstallcmd(function (package, batchcmds)
        batchcmds:rm(package:installdir("etc"))
    end)
```

Hooks available: `before_installcmd` / `on_installcmd` / `after_installcmd`, same for `uninstall`, `build`, `package`. Use `batchcmds:*` for commands that should end up in the installer script (portable across formats), or plain `os.*` for configure-time work.

## 7. Format-specific knobs

### NSIS

```lua
set_formats("nsis")
set_nsis_displayicon("res/icon.ico")
-- also set_prefixdir, add_components, etc.
```

### RPM / deb

```lua
add_buildrequires("gcc-c++", "cmake")               -- build-time deps
```

Version / maintainer / description are picked up from the xpack metadata automatically.

### WiX (MSI)

Same API surface as NSIS; WiX uses `set_title`, `set_company`, `set_nsis_displayicon` (re-used as MSI icon).

## 8. CLI reference

```bash
xmake pack                           # pack all xpack() entries in default format
xmake pack -f zip                    # force format
xmake pack -f "zip,deb" myapp        # multiple formats, one package
xmake pack myapp                     # one package only
xmake pack --autobuild=y             # rebuild targets before packing (default)
xmake pack --autobuild=n             # assume targets already built
xmake pack -o /tmp/out               # output directory
xmake pack -vD                       # verbose + diagnosis
```

Output locations:

- Default: `build/xpack/<xpack-name>/<xpack-name>-<version>-<plat>-<arch>.<ext>`
- With `-o`: under the specified directory.

## 9. Typical workflows

### Ship a cross-platform zip + installer

```lua
xpack("myapp")
    set_version("1.0.0")
    set_formats("zip", "nsis", "deb", "targz")
    set_homepage("https://example.com")
    set_license("MIT")
    add_targets("myapp")
    set_prefixdir("myapp-$(version)")
```

```bash
xmake f -p windows -m release && xmake && xmake pack -f "zip,nsis"
xmake f -p linux   -m release && xmake && xmake pack -f "targz,deb"
```

### Source tarball for distribution

```lua
xpack("myapp-src")
    set_formats("srczip", "srctargz")
    add_sourcefiles("(src/**)", "(include/**)", "xmake.lua", "README.md", "LICENSE")
```

```bash
xmake pack myapp-src
```

### Reproduce a Linux-distribution package

```lua
xpack("myapp")
    set_formats("deb")
    set_maintainer("Ops Team <ops@example.com>")
    add_buildrequires("libssl-dev")
    add_targets("myapp")
```

```bash
xmake pack -f deb
```

## Pitfalls

- **Forgetting `includes("@builtin/xpack")`.** `xpack(...)` is undefined without it — "attempt to call a nil value".
- **`xmake pack` before `xmake`.** With `--autobuild=y` (default) this is fine, but if you disable autobuild make sure targets are built first.
- **Version mismatch.** If `xpack:set_version` and `target:set_version` disagree, the xpack value wins. Keep them in sync or drop one.
- **NSIS / WiX on non-Windows.** Cross-packing needs the appropriate tooling installed — NSIS works from Linux/macOS with `makensis`, WiX generally doesn't. Install the binaries or package on the native OS.
- **`add_installfiles` paths.** Globs (`assets/**`) are relative to the project root, not the target. Paths inside the archive are set by `prefixdir`.
- **Components ignored on basic archives.** `zip`/`targz` flatten components into a single archive. Components are only honored by NSIS/WiX/deb/rpm.

## When to branch out

- Packaging *library* code for reuse by other xmake projects → `xmake-private-packages`
- Plain install via `xmake install` (no archive) → `xmake-commands`
- Running xpack from CI with binary artifacts → `xmake-build-optimization`
