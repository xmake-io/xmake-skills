# project-config

The core declarative APIs that make up an `xmake.lua` — targets, options, packages, rules, capability probes, link control, policies. If the task is "configure the build", it's in here.

| Skill | Topic |
| --- | --- |
| [xmake-targets](./xmake-targets/) | `target()`, kinds, `add_files`, deps, flags |
| [xmake-options](./xmake-options/) | `option()` declarations, `has_config`, config.h generation |
| [xmake-packages](./xmake-packages/) | `add_requires` / `add_packages` dependency integration |
| [xmake-rules](./xmake-rules/) | Built-in rules (`mode.*`, `c++.unity_build`, …) and custom rules |
| [xmake-feature-check](./xmake-feature-check/) | `has_cfuncs`, `check_cxxsnippets`, capability probes |
| [xmake-link-order](./xmake-link-order/) | `add_linkorders`, `add_linkgroups`, `--whole-archive` |
| [xmake-policy](./xmake-policy/) | `set_policy` feature toggles — sanitizers, LTO, modules, etc. |
