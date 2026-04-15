---
name: xmake-color-output
description: Use when emitting colored/styled output from `xmake.lua` or custom tasks/plugins via `cprint` / `cprintf`, or when the user wants to disable colors (`XMAKE_COLORTERM=nocolor`, `--theme=plain`). Covers the `${...}` color tag syntax, all color names, truecolor, emoji, and the disable knobs.
---

# cprint, Color Tags & Disabling Color

`cprint` / `cprintf` are xmake's colored terminal printers. They look like `print`/`printf` but accept inline style tags:

```lua
cprint("${bright green}hello ${clear}xmake")
```

Available in any script-domain code: `on_load`, `on_build`, `after_build`, `on_run`, `on_test`, task bodies, external modules imported via `import()`. Not available in the description domain (top-level `set_*`/`add_*`) — use `print` there instead, and only for debugging.

## Basic usage

```lua
cprint("hello")                       -- plain text, like print
cprint("${red}error")                 -- red text
cprint("${bright red}ERROR")          -- bold red
cprintf("%d errors\n", n)             -- printf-style
print("hello, %s", name)              -- plain print; no tags, no newline-less variant
```

Formatting uses `string.format`-style substitution. `cprint` adds a trailing newline; `cprintf` does not.

## Tag syntax

Styles are wrapped in `${...}` and apply to the rest of the line (or until `${clear}`):

```lua
cprint("${red}hello ${clear}xmake")          -- only "hello " is red
cprint("${bright red underline onyellow}ALERT${clear}")
```

Multiple attributes are space-separated inside one `${...}`.

## Complete attribute list

Attributes:

| Tag | Effect |
| --- | --- |
| `reset` / `clear` / `default` | reset all style |
| `bright` | bold / highlight |
| `dim` | dark / faint |
| `underline` | underline |
| `blink` | blinking text |
| `reverse` | swap fg/bg |
| `hidden` | invisible text |

Foreground colors:

```
black  red  green  yellow  blue  magenta  cyan  white
```

Background colors (prefix `on`):

```
onblack  onred  ongreen  onyellow  onblue  onmagenta  oncyan  onwhite
```

Example:

```lua
cprint("${bright red underline onyellow}hello${clear}")
cprint("${dim cyan}note:${clear} did you mean ${bright yellow}--foo${clear}?")
```

## 24-bit true color

Since v2.1.7, if the terminal reports truecolor support xmake accepts RGB triples:

```lua
import("core.base.colors")
if colors.truecolor() then
    cprint("${255;0;0}red")                 -- foreground RGB
    cprint("${on;255;0;0}bg red${clear}")   -- background RGB
    cprint("${bright 255;0;0 underline}bold red underlined")
    cprint("${on;255;0;0 0;255;0}green-on-red${clear}")
end
```

Force-enable if your terminal is not auto-detected:

```bash
export XMAKE_COLORTERM=truecolor
```

## Emoji

```lua
cprint("build ok ${beer}")
cprint("${ok_hand} done")
```

Falls back to nothing on terminals that do not support the emoji. See the [emoji cheat sheet](http://www.emoji-cheat-sheet.com/) for all keys.

## Disabling color output

Three knobs, from least to most invasive:

### 1. Per-invocation env var

```bash
XMAKE_COLORTERM=nocolor xmake
```

Other values: `color8`, `color256`, `truecolor`. `nocolor` strips all ANSI codes.

### 2. Persistent global

```bash
export XMAKE_COLORTERM=nocolor              # shell session
```

Or in CI:

```yaml
env:
  XMAKE_COLORTERM: nocolor
```

### 3. Theme switch (covers color + emoji + progress style)

```bash
xmake g --theme=plain                       # no color, no emoji, no progress tricks
```

Use this on terminals that choke on either ANSI escapes or unicode (old Windows cmd, some pipes, some log collectors).

Revert:

```bash
xmake g --theme=default
# or
xmake g -c
```

See `xmake-theme` for the full theme list.

## Choosing `cprint` / `print` / `vprint`

| Call | When to use |
| --- | --- |
| `print(fmt, ...)` | Plain informational messages |
| `cprint(fmt, ...)` | Styled / colored output |
| `cprintf(fmt, ...)` | Styled output without trailing newline |
| `printf(fmt, ...)` | Plain output without trailing newline |
| `utils.vprint(fmt, ...)` | Only print in `-v` verbose mode |
| `utils.cprint(fmt, ...)` | Same as `cprint` but module-namespaced |
| `raise(fmt, ...)` | Error — abort with a message (uses styling) |

Prefer `cprint` for user-facing info, `vprint` for debug traces that should hide under `-v`.

## Pitfalls

- **`cprint` in description domain.** Parsed multiple times — you'll see the message repeated. Move to `on_load` or a task hook.
- **Tags that stay "on".** `${red}` affects everything that follows on the same line. Always close with `${clear}` if you only want part of the line colored.
- **Truecolor tags on an 8-color terminal.** Xmake degrades cleanly to nearest-color on older terminals, but the result may look washed out — feature-detect with `colors.truecolor()`.
- **Piping color into logs.** `cprint` still emits ANSI when stdout is a pipe. Set `XMAKE_COLORTERM=nocolor` for log files, or use `--theme=plain` in CI.
- **Emoji on Windows.** Legacy Windows cmd doesn't render them. Use `plain` theme there.
- **`print` has no format tags.** Passing `"${red}hello"` to `print` just prints the literal `${red}`. Must use `cprint`.

## When to branch out

- Picking a global theme / fixing garbled terminals → `xmake-theme`
- All env vars incl. `XMAKE_COLORTERM` → `xmake-env-vars`
- Where scripts can call `cprint` → `xmake-scripting`
