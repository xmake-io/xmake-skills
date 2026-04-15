---
name: xmake-fortran
description: Use when building Fortran projects with xmake — binary/library targets with `.f`/`.f90`/`.for` sources, gfortran or Intel Fortran (`ifort`) toolchain selection, and mixing Fortran with C interoperability.
---

# Building Fortran with Xmake

Xmake supports Fortran since v2.3.6 via `gfortran`, and Intel Fortran (`ifort`) since v2.3.8.

## 1. Minimal project

```bash
xmake create -l fortran -t console hello
cd hello
xmake run
```

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.f90")
```

Recognized extensions: `.f`, `.F`, `.f90`, `.F90`, `.f95`, `.F95`, `.f03`, `.F03`, `.f08`, `.F08`, `.for`, `.FOR`. The uppercase variants get preprocessed.

## 2. Target kinds

```lua
target("app")       set_kind("binary")
target("mylib")     set_kind("static")
target("mydyn")     set_kind("shared")
```

## 3. Compiler selection

```bash
xmake f --toolchain=gfortran              # default gfortran
xmake f --toolchain=ifort                 # Intel Fortran
xmake f --toolchain=ifx                   # Intel oneAPI LLVM-based Fortran
```

Pin a specific install with `--sdk=/opt/intel/oneapi`.

## 4. Flags

```lua
target("app")
    add_files("src/*.f90")
    add_fcflags("-O3", "-ffree-form")           -- gfortran flags
    add_ldflags("-lm")
```

`add_fcflags` = Fortran compiler flags.

## 5. Mixing Fortran with C

```lua
target("solver")
    set_kind("static")
    add_files("src/*.f90", "src/*.c")
    add_includedirs("include", {public = true})

target("app")
    set_kind("binary")
    add_files("src/main.cpp")
    add_deps("solver")
    set_languages("c++17")
```

Use `iso_c_binding` on the Fortran side and `extern "C"` on the C++ side. Xmake handles the link step — gfortran's runtime is added automatically when Fortran objects are present.

## 6. Modules (Fortran modules, not C++ modules)

Fortran `module`/`end module` produces `.mod` files. Xmake tracks them automatically for dependency ordering — list source files in `add_files` in any order; xmake scans `USE` statements.

## Pitfalls

- **Fixed-form vs free-form.** `.f`/`.for` default to fixed form (columns matter); `.f90`+ default to free form. Flag overrides work per-file if needed.
- **`.mod` leaking across targets.** Each target has its own module cache. Share modules by linking the static/shared library and letting xmake propagate the `-I` path.
- **Intel compiler flag dialect.** `ifort` and `ifx` differ from `gfortran` in many flag names. Gate with `is_toolchain`.
- **Mixing Fortran and C++ without a C intermediate.** Fortran↔C++ name mangling is messy; always bridge through a C layer.

## When to branch out

- Target basics → `xmake-targets`
- Cross-compile → `xmake-cross-compilation`
- Mixing with C/C++ → `xmake-targets`
