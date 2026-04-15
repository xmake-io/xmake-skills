---
name: xmake-csharp
description: Use when building C# projects with xmake — `.cs` sources with the Mono `mcs` or .NET `dotnet` toolchain, binary / library output, adding references, and mixing with native code via P/Invoke.
---

# Building C# with Xmake

Xmake drives the C# compiler (Mono `mcs` or modern .NET `csc`/`dotnet`) through a target model similar to C++.

## 1. Minimal project

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.cs")
```

## 2. Target kinds

```lua
target("app")       set_kind("binary")       -- .exe
target("mylib")     set_kind("shared")       -- .dll (class library)
```

## 3. Toolchain

```bash
xmake f --toolchain=mcs                      -- Mono compiler
xmake f --toolchain=dotnet                   -- .NET SDK (csc)
```

Pin a specific install with `--sdk`:

```bash
xmake f --toolchain=dotnet --sdk=/usr/share/dotnet/sdk/8.0.100
```

## 4. References

```lua
target("app")
    add_files("src/*.cs")
    add_csflags("/reference:System.dll", "/reference:System.Core.dll")
    -- Or the modern, portable form:
    add_values("cs.references", "System.dll", "System.Core.dll")
```

For NuGet / .NET SDK projects, prefer a real `.csproj` consumed via `dotnet build` — xmake's direct C# support is best for small / standalone tools.

## 5. Mixing with native code (P/Invoke)

```lua
target("nativelib")
    set_kind("shared")
    add_files("native/*.c")

target("app")
    set_kind("binary")
    add_files("src/*.cs")
    add_deps("nativelib")
```

In `Program.cs`:

```cs
using System.Runtime.InteropServices;
class Program {
    [DllImport("nativelib")]
    static extern int hello(int x);
}
```

Xmake places `nativelib.dylib`/`.so`/`.dll` next to the managed binary so the runtime loader finds it.

## 6. Flags

```lua
target("app")
    add_files("src/*.cs")
    add_csflags("-unsafe", "-nowarn:1591")
```

`add_csflags` maps to C# compiler flags.

## Pitfalls

- **`xmake` is not `dotnet build`.** Complex .csproj projects (NuGet, SDK-style projects, Razor, WPF designers) should go through `dotnet build` inside a `task(...)` instead of xmake's native C# support.
- **Missing `System.dll` references.** Modern .NET Core doesn't need them (implicit); Mono does. Test with the toolchain you'll ship on.
- **Managed DLL + native DLL path mismatch.** P/Invoke lookups fail if the native DLL isn't on `PATH` or next to the .exe. Xmake's default placement handles the common case.

## When to branch out

- Native library interop → `xmake-targets`
- Packaging a C# app → `xmake-xpack`
