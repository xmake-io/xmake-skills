# languages

Building non-C/C++ languages with xmake. Each skill covers target kinds, extensions, toolchain selection, typical flags, and interop with native code.

C and C++ are covered by the core skills under [`basics/`](../basics/), [`project-config/`](../project-config/), [`toolchains/`](../toolchains/), and [`packages/`](../packages/) — there is no dedicated `xmake-cpp` skill because it's the default everywhere.

| Skill | Language | Notes |
| --- | --- | --- |
| [xmake-rust](./xmake-rust/) | Rust | Cargo integration, cxxbridge interop |
| [xmake-go](./xmake-go/) | Go | Modules, cross-compile, CGO |
| [xmake-swift](./xmake-swift/) | Swift | C++/Objective-C interop, Apple platforms |
| [xmake-objc](./xmake-objc/) | Objective-C / Objective-C++ | Cocoa frameworks, `.app` bundles |
| [xmake-dlang](./xmake-dlang/) | D | DUB packages, DMD/LDC/GDC |
| [xmake-fortran](./xmake-fortran/) | Fortran | gfortran / Intel Fortran, C interop |
| [xmake-cuda](./xmake-cuda/) | CUDA | `.cu` sources, `sm_*` gencodes, device linking |
| [xmake-zig](./xmake-zig/) | Zig | Native cross-compile, `zig cc` as C compiler |
| [xmake-nim](./xmake-nim/) | Nim | Backend selection (c/cpp/objc/js), Nim flags |
| [xmake-pascal](./xmake-pascal/) | Pascal | FreePascal, binary and shared library targets |
| [xmake-vala](./xmake-vala/) | Vala | `add_rules("vala")`, glib, VAPI packages |
| [xmake-csharp](./xmake-csharp/) | C# | Mono `mcs` / .NET `dotnet`, P/Invoke |
| [xmake-kotlin](./xmake-kotlin/) | Kotlin/Native | Not JVM — for JVM/Android use Gradle |
