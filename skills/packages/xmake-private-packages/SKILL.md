---
name: xmake-private-packages
description: Use when distributing C/C++ libraries *inside a company or team* — packaging a library with `xmake package`, creating a private package repository (local directory or git), writing package recipes for source/binary distribution, and consuming them via `add_repositories` + `add_requires`. For shipping end-user installers (zip/nsis/deb), see `xmake-xpack`.
---

# Private Packages & Repository Distribution

This skill covers distributing C/C++ libraries as xmake **packages** — i.e. consumable via `add_requires("mylib")` in a downstream project. Three scenarios:

1. **Local package** — `xmake package` emits a directory of prebuilt binaries + a tiny `xmake.lua` repo you can share over NFS/file share.
2. **Remote source package** — a private git repo containing recipes that fetch source (usually from another private git repo) and build on demand.
3. **Remote binary package** — private git repo of recipes that download a pre-built tarball (no source leak, fast install).

All three reuse xmake's standard `add_requires` / `add_packages` integration on the consumer side — you just swap the repository URL.

## 1. Prepare the library project

```lua
-- library project: xmake.lua
add_rules("mode.debug", "mode.release")

target("foo")
    set_kind("static")                           -- or "shared"
    set_version("1.0.0")
    add_files("src/*.cpp")
    add_headerfiles("src/foo.h")                 -- install to include/foo.h
    add_headerfiles("src/foo_private.h", {prefixdir = "foo/internal"})
    add_includedirs("include", {public = true})  -- expose to consumers
```

Key points:

- `add_headerfiles(...)` marks headers for installation. `prefixdir` adds a subdirectory under `include/`.
- `add_includedirs(..., {public = true})` is what makes downstream targets actually see the headers.
- `set_version` here flows into the package metadata.

## 2. Local package distribution

### Produce the package

Run inside the library project root:

```bash
xmake package                    # local package (default)
# equivalent:
xmake package -f local
```

Output:

```
build/packages/
└── f/
    └── foo/
        ├── macosx/x86_64/release/
        │   ├── include/foo.h
        │   └── lib/libfoo.a
        └── xmake.lua            # the generated recipe
```

`build/packages/` is already a valid xmake package repository. You can copy it anywhere — shared drive, S3, local tarball — and consume it.

### Consume it

In another project's `xmake.lua`:

```lua
-- point at the local package tree
add_repositories("local-repo /nfs/shared/foo-packages")

add_requires("foo")

target("bar")
    set_kind("binary")
    add_files("src/*.cpp")
    add_packages("foo")
```

`add_repositories("<name> <path-or-url>")` registers a repo. Xmake searches every registered repo for `foo` and uses the first match.

**Benefits of local packages:**

- No git needed.
- Binary is prebuilt — instant integration.
- Multi-arch/mode support: run `xmake package` on each target platform and merge the trees.

## 3. Remote package via private git repo

### Generate a recipe template

```bash
xmake package -f remote
```

This creates `packages/f/foo/xmake.lua` you can edit into a proper recipe.

### Source-distribution recipe

```lua
-- packages/f/foo/xmake.lua
package("foo")
    set_description("The foo library")
    set_license("Apache-2.0")

    add_urls("git@github.com:mycompany/foo.git")
    add_versions("1.0.0", "<commit-sha-or-tag>")

    on_install(function (package)
        local configs = {}
        if package:config("shared") then
            configs.kind = "shared"
        end
        import("package.tools.xmake").install(package, configs)
    end)

    on_test(function (package)
        assert(package:has_cxxfuncs("foo_init", {includes = "foo.h"}))
    end)
package_end()
```

The recipe lives in its own `my-repo` git repo alongside any other recipes:

```
my-repo/
└── packages/
    ├── f/
    │   └── foo/
    │       └── xmake.lua
    └── b/
        └── bar/
            └── xmake.lua
```

### Binary-distribution recipe

If you don't want consumers to see source or recompile:

```lua
package("foo")
    set_description("The foo library (binary)")

    -- a server hosting pre-built tarballs
    add_urls("https://dist.example.com/foo/foo-$(version).tar.gz")
    add_versions("1.0.0", "<sha256>")

    on_install(function (package)
        -- the tarball already contains include/ and lib/
        os.cp("include", package:installdir())
        os.cp("lib",     package:installdir())
    end)

    on_test(function (package)
        assert(package:has_cxxfuncs("foo_init", {includes = "foo.h"}))
    end)
package_end()
```

Upload the tarball (produced by `xmake package`, `xmake pack -f targz`, or any other tool) to a server; the recipe pulls it by URL.

### Consume from a private git repo

```lua
add_repositories("my-repo git@github.com:mycompany/my-repo.git")
add_requires("foo 1.0.0")

target("app")
    add_files("src/*.cpp")
    add_packages("foo")
```

First build clones the recipe repo into `~/.xmake/repositories/my-repo`, then installs the package per the recipe.

## 4. In-project package repo

For a one-off private package used only within a single project, keep the repo inside the project itself:

```
myproject/
├── packages/
│   └── f/
│       └── foo/
│           └── xmake.lua
├── src/
│   └── main.cpp
└── xmake.lua
```

```lua
-- myproject/xmake.lua
add_repositories("project-local packages")       -- relative path
add_requires("foo")

target("myproject")
    add_files("src/*.cpp")
    add_packages("foo")
```

No external git, no shared tree — the recipe lives next to the code.

## 5. Managing private repos globally

Add/remove global repo registrations:

```bash
xrepo add-repo   mycompany git@github.com:mycompany/my-repo.git
xrepo rm-repo    mycompany
xrepo list-repo
xrepo update-repo                           # pull latest recipes
```

Once globally registered, any project can `add_requires("foo")` without its own `add_repositories` line.

## 6. Testing a recipe before publishing

Use `xmake-repo-testing`'s workflow inside your private repo:

```bash
cd my-repo
xmake l scripts/test.lua -vD --shallow foo
```

Catches recipe bugs before consumers hit them. See the `xmake-repo-testing` skill.

## 7. C++ modules packages

For pure C++20 module libraries, set `moduleonly` so xmake skips traditional linking:

```lua
-- library project
target("foo")
    set_kind("moduleonly")
    add_files("src/*.mpp")
```

```lua
-- recipe
package("foo")
    set_kind("library", {moduleonly = true})
    set_sourcedir(path.join(os.scriptdir(), "src"))
    on_install(function (package)
        import("package.tools.xmake").install(package, {})
    end)
```

Consumer:

```lua
add_requires("foo")

target("app")
    set_languages("c++20")
    set_policy("build.c++.modules", true)
    add_packages("foo")
```

See `xmake-cxx-modules` for the build side.

## Pitfalls

- **Forgetting `add_headerfiles`.** Libraries without declared header files install only binaries — consumers get link errors because the include path is missing.
- **`add_includedirs` private.** Without `{public = true}`, downstream targets don't see the include path even after `add_packages("foo")`.
- **Mixing local + remote repos with same package name.** Xmake picks the first match. Order `add_repositories` deliberately.
- **Version pinning missed.** `add_requires("foo")` gets whatever version the recipe's `add_versions` lists last. Pin with `"foo 1.0.0"` when the consumer needs determinism.
- **Stale repo clones.** `xrepo update-repo` before debugging "why isn't my new recipe showing up?".
- **Binary package + cross-compile.** A binary recipe is pinned to the build host's arch. Ship separate tarballs per platform/arch and use `add_urls` conditionally or multiple `add_versions` with arch filters.
- **Private git auth in CI.** Recipe repos via `git@...` need SSH keys in CI. Use HTTPS with a token or pre-clone `~/.xmake/repositories/` in CI setup.

## When to branch out

- Packaging end-user installers (zip/nsis/deb/rpm) → `xmake-xpack`
- Testing package recipes (`xmake l scripts/test.lua`) → `xmake-repo-testing`
- C++20 modules build-side configuration → `xmake-cxx-modules`
- Consuming packages (add_requires / add_packages) → `xmake-packages`
- Repository management commands → `xrepo-cli`
