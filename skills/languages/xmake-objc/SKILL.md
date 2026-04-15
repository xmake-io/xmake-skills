---
name: xmake-objc
description: Use when building Objective-C or Objective-C++ projects with xmake — `.m` / `.mm` sources, Cocoa / Foundation frameworks via `add_frameworks`, mixing with C++, and targeting macOS / iOS / tvOS.
---

# Building Objective-C / Objective-C++ with Xmake

Objective-C (`.m`) and Objective-C++ (`.mm`) are first-class in xmake. Primary use case: macOS / iOS / tvOS apps, Cocoa frameworks, and mixed C++ / Objective-C projects.

## 1. Minimal Objective-C project

```lua
add_rules("mode.debug", "mode.release")

target("hello")
    set_kind("binary")
    add_files("src/*.m")
    add_frameworks("Foundation")
```

```objc
// src/main.m
#import <Foundation/Foundation.h>
int main() {
    @autoreleasepool {
        NSLog(@"hello from xmake");
    }
    return 0;
}
```

## 2. Objective-C++ (mixing with C++)

Use `.mm` extension and set the C++ language level:

```lua
target("app")
    set_kind("binary")
    set_languages("c++17")
    add_files("src/*.mm", "src/*.cpp")
    add_frameworks("Foundation", "AppKit")
```

`.mm` files compile with the Objective-C++ front-end — you can freely mix `@interface`/`@implementation` with C++ classes and templates.

## 3. Frameworks

```lua
target("app")
    add_frameworks("Foundation", "Cocoa", "AppKit", "Metal", "MetalKit", "CoreGraphics")
    add_frameworkdirs("/Library/Frameworks", "$(projectdir)/vendor/frameworks")
```

- `add_frameworks(...)` — system or user frameworks (equivalent of `-framework <name>`).
- `add_frameworkdirs(...)` — additional search paths for non-system frameworks.

## 4. Flags

```lua
target("app")
    add_mflags("-fobjc-arc")                    -- Objective-C flag (.m)
    add_mxxflags("-fobjc-arc")                  -- Objective-C++ flag (.mm)
```

`add_mflags` / `add_mxxflags` target `.m` / `.mm` respectively; `add_cxflags` still flows through to both.

ARC is the default in modern Xcode; include `-fobjc-arc` explicitly if you want to be sure.

## 5. Apple platform targets

```bash
xmake f -p macosx  -a arm64
xmake f -p iphoneos -a arm64
xmake f -p iphonesimulator -a arm64
xmake f -p appletvos -a arm64
xmake f -p watchos -a arm64
xmake
```

Pin SDK version and minimum deployment:

```bash
xmake f -p iphoneos --xcode_sdkver=17.2 --target_minver=14.0
```

## 6. Application bundle

For a proper `.app` bundle on macOS:

```lua
target("myapp")
    add_rules("xcode.application")               -- produces .app
    add_files("src/*.m", "res/*.storyboard", "res/*.xcassets")
    add_files("res/Info.plist")
    add_frameworks("Foundation", "AppKit")
```

iOS app:

```lua
target("myapp")
    add_rules("xcode.application")
    set_plat("iphoneos")
    add_files("src/*.m", "res/Info.plist")
    add_frameworks("Foundation", "UIKit")
```

`xcode.application` is a built-in rule that produces a proper bundle with Info.plist, code signing hooks, etc.

## 7. Consuming Objective-C from Swift

See `xmake-swift` — Swift interop picks up Objective-C headers via `set_values("swift.interop", "objc")`.

## 8. Pure C ↔ Objective-C

Objective-C is a strict superset of C, so a `.m` file can `#include <stdio.h>` freely. Pure C headers compile inside `.m`/`.mm` files without extra flags.

## Pitfalls

- **`.mm` without `set_languages("c++XX")`.** Defaults to C++98 which is almost never what you want. Set it explicitly.
- **System frameworks vs SDK version.** `add_frameworks("SwiftUI")` fails on old macOS SDKs. Pin with `--xcode_sdkver=` or gate with `is_plat` + minver.
- **Missing `-fobjc-arc` on old code.** Pre-ARC code uses `retain`/`release`; modern code uses ARC. Don't mix arbitrarily.
- **Info.plist missing from bundle.** `xcode.application` needs an `Info.plist` in `add_files` or it produces a broken bundle.
- **Cross-compile to iOS without Xcode.** Only possible on macOS — iOS SDK is Xcode-only.

## When to branch out

- Apple-platform toolchain / SDK → `xmake-cross-compilation`, `xmake-toolchains`
- Swift interop → `xmake-swift`
- Target basics → `xmake-targets`
- Packaging a .app bundle as .dmg / .ipa → `xmake-xpack`
