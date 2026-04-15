---
name: xmake-debug-package-source
description: Use when a package from `add_requires` / xmake-repo fails to build or misbehaves, and you need to debug its source — editing the upstream files in place without patches, running the recipe against a local source tree with `-d <dir>`, stepping through `on_install`, extracting the patch once fixed, and remote debugging via `--remote`.
---

# Debugging Package Source Code

Packages installed via `add_requires` / xrepo fail for many reasons — upstream bug on your toolchain, outdated patch, missing flag. This skill is about the iteration loop for **fixing a broken package recipe or upstream source** without losing work between attempts.

Default recipe flow:

```
add_requires("foo") → fetch tarball → apply patches → on_install → on_test
```

Each time the recipe runs, xmake blows away the extracted source and re-applies patches. That's fine for stable packages, terrible for debugging — every change you make to the unpacked source is lost on the next run.

## 1. Fast-path: `-d <local-source-dir>` with `scripts/test.lua`

Inside an `xmake-repo` clone (your own or the official one):

```bash
xmake l scripts/test.lua -vD --shallow -d /tmp/zlib-1.2.11 zlib
```

- `-d <dir>` — tells the test script to **use your local source directory** instead of re-downloading and re-extracting. Your edits persist across runs.
- `--shallow` — don't rebuild transitive dependencies (see `xmake-repo-testing`).
- `-vD` — verbose + tracebacks.

### Workflow

```bash
# 1. Download & unpack once
curl -LO https://zlib.net/zlib-1.2.11.tar.gz
tar xf zlib-1.2.11.tar.gz -C /tmp
cd xmake-repo

# 2. First run — sees what fails
xmake l scripts/test.lua -vD --shallow -d /tmp/zlib-1.2.11 zlib
# ... some configure step blows up ...

# 3. Edit /tmp/zlib-1.2.11/configure to fix it
vim /tmp/zlib-1.2.11/configure

# 4. Re-run — your edit is still there
xmake l scripts/test.lua -vD --shallow -d /tmp/zlib-1.2.11 zlib

# 5. Iterate until it builds + on_test passes
```

Without `-d`, step 4 would re-extract the tarball and wipe your edit. This is the difference between "debug in 5 minutes" and "debug in an afternoon".

## 2. Turning edits into a patch

Once you've fixed the bug locally, generate a patch for the recipe:

```bash
cd /tmp/zlib-1.2.11
tar xf ~/Downloads/zlib-1.2.11.tar.gz -C /tmp/zlib-1.2.11.orig --strip-components=1
diff -urN /tmp/zlib-1.2.11.orig /tmp/zlib-1.2.11 > /path/to/xmake-repo/packages/z/zlib/patches/my-fix.patch
```

Easier if the source directory is a git repo:

```bash
cd /tmp/zlib-1.2.11
git init && git add -A && git commit -m "upstream"
# make edits
git diff > /path/to/xmake-repo/packages/z/zlib/patches/1.2.11/fix.patch
```

Then reference it from the recipe:

```lua
-- packages/z/zlib/xmake.lua
package("zlib")
    ...
    add_patches("1.2.11", path.join(os.scriptdir(), "patches/1.2.11/fix.patch"),
                "<sha256-of-patch-file>")
```

Compute the sha256 with `xmake l hash.sha256 path/to/patch.patch` or `shasum -a 256 patch.patch`.

## 3. Debugging `on_install` itself

If the failure is in the recipe's Lua logic (not the upstream source), sprinkle `print` / `cprint`:

```lua
on_install(function (package)
    print("package install dir: %s", package:installdir())
    print("current configs: %s", os.args(table.keys(package:configs() or {})))
    print("build kind: %s", package:is_library() and "lib" or "bin")
    import("package.tools.cmake").install(package, {
        "-DBUILD_SHARED_LIBS=" .. (package:config("shared") and "ON" or "OFF"),
        "-DBUILD_TESTING=OFF"
    })
end)
```

With `-vD`, every `os.exec` the recipe runs shows up, and any Lua error prints a traceback pointing at the recipe line number.

### Useful `package` instance methods

Inside `on_install` / `on_test`, the `package` object exposes:

```lua
package:name()             -- "zlib"
package:version()          -- version instance (package:version():shortstr() for a string)
package:plat()             -- "macosx" | "linux" | ...
package:arch()             -- "x86_64" | "arm64" | ...
package:is_plat("linux")   -- convenience
package:is_cross()         -- cross-compiling?
package:config("shared")   -- recipe option value
package:configs()          -- all configs
package:installdir()       -- destination dir
package:installdir("lib")  -- subdir inside it
package:sourcedir()        -- unpacked source dir (after fetch)
package:buildhash()        -- hash identifying this build
package:dep("zlib")        -- another package the recipe depends on
package:has_cfuncs("foo", {includes = "foo.h"})    -- for on_test
```

## 4. Debug the installed package's binaries

After `on_install` succeeds, the installed tree lives under:

```
~/.xmake/packages/<a>/<name>/<version>/<hash>/
├── include/
├── lib/
├── bin/
└── xmake.lua      (metadata)
```

Inspect it:

```bash
xmake require --info zlib
# -> installdir, cachedir, fetchinfo, etc.

ls ~/.xmake/packages/z/zlib/1.2.11/*/
nm ~/.xmake/packages/z/zlib/1.2.11/*/lib/libz.a | head
```

If the binary is wrong, you're back to step 1 — fix the source, re-run with `-d`.

## 5. Force-rebuild a cached package

Xmake aggressively caches package installs. If a recipe change seems to have no effect:

```bash
xmake require --force zlib                         # project context
xrepo install --force zlib                         # standalone
rm -rf ~/.xmake/packages/z/zlib                    # nuclear
```

`--force` re-runs `on_install` and its test.

## 6. Interactive Lua inspection

Load the package in an xmake Lua REPL:

```bash
xmake l
> import("core.package.package")
> local pkg = package.load("zlib")
> print(pkg:installdir())
> print(pkg:version_str())
```

Or run a script fragment:

```bash
xmake l -c '
import("core.package.package")
local pkg = package.load("zlib")
for k, v in pairs(pkg:configs() or {}) do print(k, v) end
'
```

## 7. Remote debugging

If the failure only reproduces on another platform (Linux on your Mac, Windows on Linux), use the remote service:

```bash
# on the remote machine:
xmake service                      # start server

# locally:
cd xmake-repo
xmake service --connect
xmake l scripts/test.lua -vD --shallow --remote -d /tmp/zlib-1.2.11 zlib
```

The `-d` directory is synced to the server, the build happens there, and output streams back to you. See `xmake-remote-compilation`.

## 8. Common scenarios

### Patch works on your box but `xmake l scripts/test.lua` still downloads fresh source

Forgot `-d`. The test script only reuses your local tree when explicitly told.

### Recipe calls `import("package.tools.cmake").install(package, {...})` and fails

Read what cmake sees:

```lua
on_install(function (package)
    import("package.tools.cmake").install(package, {
        "-DCMAKE_VERBOSE_MAKEFILE=ON",        -- see actual compile commands
        "-DBUILD_TESTING=OFF"
    })
end)
```

Or drop to raw `os.vrunv` for total control:

```lua
on_install(function (package)
    os.cd(package:sourcedir())
    os.vrunv("cmake", {"-B", "build", "-G", "Ninja", "-DBUILD_TESTING=OFF"})
    os.vrunv("cmake", {"--build", "build", "--target", "install"})
    os.cp("build/install/*", package:installdir())
end)
```

### `on_test` fails but the lib seems fine

Run the test compile manually:

```bash
find ~/.xmake/cache/packages -name "test.cpp" -path "*zlib*"
# look at what xmake tried to compile; run it by hand with the right -I / -L / -l
```

Or make `on_test` simpler to isolate:

```lua
on_test(function (package)
    assert(os.isfile(path.join(package:installdir("lib"), "libz.a")))
end)
```

## Pitfalls

- **Editing `~/.xmake/packages/<pkg>/<hash>/` directly.** That's the installed tree — xmake rebuilds it from source on any recipe/source change. Edits there are ephemeral. Work in the source dir you pointed `-d` at.
- **Missing `--shallow`.** Without it, every dep is rebuilt from source on each iteration — glacial.
- **Recipe patches re-applied over your edits.** If the recipe declares `add_patches`, xmake applies them after extracting source. Either remove the patch while debugging or commit your edits to a git branch on top and regenerate the patch at the end.
- **Hash mismatch after changing the tarball URL.** `add_versions` pins a sha256; changing the URL/version without updating the hash makes xmake refuse.
- **Confusing `on_install` errors with upstream errors.** An error from `import("package.tools.cmake").install` may come from cmake itself — read up the stack carefully.
- **Stale `~/.xmake/packages` after toolchain switch.** A package built with gcc-11 won't re-run for gcc-12 automatically unless the hash changes. `--force` or delete the hash dir.

## When to branch out

- Writing / submitting a new recipe → `xmake-repo-testing`
- Distributing packages privately → `xmake-private-packages`
- Using packages from a consuming project → `xmake-packages`
- Remote service setup → `xmake-remote-compilation`
- General xmake troubleshooting (non-package) → `xmake-troubleshooting`
