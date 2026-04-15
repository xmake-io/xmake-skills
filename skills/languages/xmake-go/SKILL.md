---
name: xmake-go
description: Use when building Go projects with xmake — binary/library targets with `.go` sources, go module integration, cross-compiling Go programs for other OS/arch, and mixing Go with C via cgo.
---

# Building Go with Xmake

Xmake builds Go via the Go toolchain (`go build` under the hood, wrapped in xmake's target model). Useful if your project mixes Go with C/C++/Rust and you want a single build tool.

## 1. Minimal project

```bash
xmake create -l go -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.go")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")       -- executable
target("mylib")     set_kind("static")       -- Go archive
target("mydyn")     set_kind("shared")       -- Go shared lib (CGO-compatible)
```

## 3. Go modules

Keep a normal `go.mod` at the project root. Xmake picks it up automatically:

```
myproj/
├── xmake.lua
├── go.mod
├── go.sum
└── src/
    └── main.go
```

Dependencies declared in `go.mod` (`require github.com/... vX.Y.Z`) resolve through the Go toolchain as usual — xmake does not second-guess them.

## 4. Cross-compile

Go is famously easy to cross-compile, and xmake exposes it through standard flags:

```bash
xmake f -p windows -a x86_64
xmake                                    # builds a .exe from macOS/Linux

xmake f -p linux   -a arm64
xmake

xmake f -p macosx  -a arm64               # Apple Silicon
xmake
```

Xmake sets `GOOS` / `GOARCH` for you. Supported plats map to Go's naming automatically.

## 5. CGO (calling C from Go)

Keep `import "C"` in your `.go` files as usual. When building, add any C dependencies the Go code calls into:

```lua
target("app")
    set_kind("binary")
    add_files("src/*.go", "src/*.c")
    add_includedirs("include")
    add_links("foo")
```

Xmake compiles the `.c` files and links them alongside the Go objects.

## 6. Toolchain selection

```bash
xmake f --toolchain=go
xmake f --toolchain=go --sdk=/opt/go-1.22
```

`--sdk` pins a specific Go install; otherwise `go` on PATH is used.

## 7. Flags

```lua
target("app")
    add_files("src/*.go")
    add_gcflags("-N", "-l")               -- passed to `go tool compile`
    add_ldflags("-s -w")                  -- passed to `go tool link` (stripping)
```

`add_gcflags` is Go-specific (go **c**ompile flags). `add_ldflags` maps to `go tool link -ldflags`.

## 8. Tests

```lua
target("app_test")
    set_kind("binary")
    add_files("tests/*_test.go")
    add_tests("default")
```

Or run Go's native test runner as an xmake task:

```lua
task("gotest")
    set_menu { usage = "xmake gotest", description = "Run go test" }
    on_run(function () os.exec("go test ./...") end)
```

## Pitfalls

- **Editing `go.mod` after `xmake f`.** Run `xmake f -c` to re-resolve.
- **CGO disabled in cross-compile.** `CGO_ENABLED=0` is often required for pure-Go cross builds. Set it via env or `os.setenv` in `on_load`.
- **Mixing `.go` and `.cpp` (not `.c`) in one target.** Use `.c` for the C side; Go's CGO expects C, not C++.
- **`GOFLAGS` in env.** Sometimes overrides what xmake sets — `unset GOFLAGS` if you hit weird behavior.

## When to branch out

- CGO / mixing with C++ → `xmake-targets`
- Cross-compile flag plumbing → `xmake-cross-compilation`
- Packaging the built binary → `xmake-xpack`
