---
name: xmake-packages
description: Use when adding third-party C/C++ dependencies to an Xmake project — via `add_requires` / `add_packages`, configuring package options, pinning versions, or using `xrepo` from the command line.
---

# Xmake Packages

Xmake has a built-in package manager (xrepo) that fetches and builds C/C++ dependencies and wires them into your targets.

## Adding a dependency

```lua
add_requires("fmt 10.2.1", "spdlog")

target("app")
    set_kind("binary")
    add_files("src/*.cpp")
    add_packages("fmt", "spdlog")
```

`add_requires` declares *what* you need (and optionally a version). `add_packages` attaches the package to a specific target, adding its includes, links, and defines automatically.

## Version selectors

```lua
add_requires("zlib 1.3.x")        -- semver range
add_requires("openssl >=3.0")
add_requires("cmake master")      -- branch
add_requires("boost 1.84.0")
```

## Package options

```lua
add_requires("boost", {configs = {regex = true, system = true, shared = false}})
add_requires("ffmpeg", {configs = {shared = true, gpl = false}})
```

Options map to the package's own configurable features — check the package with `xrepo info <name>`.

## Optional and system packages

```lua
add_requires("mysql", {optional = true})          -- build continues if missing
add_requires("zlib", {system = true})             -- only use the system copy
add_requires("zlib", {system = false})            -- force xrepo to build it
```

## Conditional / platform-specific

```lua
if is_plat("linux") then
    add_requires("libuuid")
end

add_requires("directxtk", {plat = "windows"})
```

## Private repositories

```lua
add_repositories("my-repo https://github.com/me/my-xmake-repo.git")
add_requires("mypkg")
```

## CLI — xrepo

```bash
xrepo search fmt
xrepo info fmt
xrepo install "fmt 10.2.1"
xrepo env -b fmt bash          # shell with fmt on PATH/PKG_CONFIG_PATH
```

## Troubleshooting

- Force a rebuild of a package: `xmake require --force fmt`
- Clear the package cache: `xmake require --clean`
- Show resolved packages: `xmake require --info fmt`

## When to branch out

- Exposing package usage behind an option → `xmake-options`
- Cross-compiled packages → `xmake-toolchains`
