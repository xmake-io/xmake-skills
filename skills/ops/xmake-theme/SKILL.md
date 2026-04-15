---
name: xmake-theme
description: Use when switching xmake's output theme (colors, progress style, emoji) or fixing garbled/unreadable terminal output — `xmake g --theme=...`. Built-in themes: default, ninja, emoji, dark, light, plain, powershell.
---

# Xmake Theme Style

Xmake's terminal output can be re-themed globally. Themes control color palette, the progress bar style, and whether emoji are used. Configured per-machine via `xmake g`.

## Switch theme

```bash
xmake g --theme=default       # default: colorful, rolling progress
xmake g --theme=ninja         # ninja-style single-line progress bar
xmake g --theme=emoji         # uses emoji in place of colors
xmake g --theme=dark          # darker palette for very light terminals
xmake g --theme=light         # light palette for very dark terminals
xmake g --theme=plain         # strip all color/emoji — fixes garbled terminals
xmake g --theme=powershell    # tuned for Windows PowerShell's odd palette
```

Reset to default:

```bash
xmake g -c                    # reset all global config
xmake g --theme=default
```

## When to pick which

| Situation | Theme |
| --- | --- |
| Default dark terminal, you just want it to work | `default` |
| Terminal shows garbled ANSI escapes (old Windows cmd, dumb terminals) | `plain` |
| Warnings invisible because bg is yellow | `dark` |
| Text overlap on dark backgrounds | `light` |
| Magenta looks wrong (PowerShell) | `powershell` |
| Want persistent single-line progress like ninja | `ninja` |
| Fun / emoji UI | `emoji` |

## Color control without switching themes

If you just want to toggle color output, the `XMAKE_COLORTERM` env var is simpler:

```bash
export XMAKE_COLORTERM=nocolor       # disable all color
export XMAKE_COLORTERM=color8        # 8 colors
export XMAKE_COLORTERM=color256      # 256 colors
export XMAKE_COLORTERM=truecolor     # 24-bit true color
```

Or globally:

```bash
xmake g --theme=plain                # full disable — same effect as nocolor
```

## Pitfalls

- **Theme is global.** There is no per-project theme. If you need a different look for CI, set `XMAKE_COLORTERM=nocolor` in the CI env rather than switching themes.
- **Powershell garbage on Windows.** Magenta renders wrong on legacy PowerShell. `xmake g --theme=powershell` fixes it.
- **`--theme=plain` also hides emoji.** Intentional — use it when terminals choke on both ANSI and unicode.
- **Old Windows cmd still broken with any theme.** Use `plain`, or switch to Windows Terminal / PowerShell 7.

## When to branch out

- Emitting your own colored output from scripts/tasks → `xmake-color-output`
- Debugging why colors look wrong at all → `xmake-troubleshooting`
