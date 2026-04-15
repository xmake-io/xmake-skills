# Skills

Each leaf directory under this tree is a self-contained [Agent Skill](https://www.anthropic.com/news/agent-skills) for one area of Xmake. Skills are grouped by **when an agent would reach for them** — the category you land in is a hint about the kind of task you're working on, not a strict technical taxonomy.

## Layout

```
skills/
├── <category>/
│   ├── README.md           # category index (one-line summaries)
│   └── <skill-name>/
│       ├── SKILL.md        # frontmatter + instructions
│       ├── references/     # optional: supporting docs (loaded on demand)
│       └── scripts/        # optional: helper scripts
```

Each `SKILL.md` has a frontmatter block with `name` and `description` — the agent reads the description to decide whether to load the full instructions.

## Categories

### [basics/](./basics/)

Getting started — installation, first project, idiomatic style, templates.

- xmake-basics, xmake-style, xmake-templates

### [project-config/](./project-config/)

Core `xmake.lua` APIs — targets, options, packages, rules, probes, link control, policies.

- xmake-targets, xmake-options, xmake-packages, xmake-rules
- xmake-feature-check, xmake-link-order, xmake-policy

### [cli/](./cli/)

Running xmake from the command line and exporting to other build systems.

- xmake-commands, xmake-project-generator, xmake-trybuild

### [toolchains/](./toolchains/)

Compiler selection, cross-compilation, C++20 modules.

- xmake-toolchains, xmake-cross-compilation, xmake-cxx-modules

### [packages/](./packages/)

C/C++ packages — `xrepo`, recipes, private repos, network acceleration.

- xrepo-cli, xrepo-env, xmake-network
- xmake-repo-testing, xmake-debug-package-source, xmake-private-packages

### [packaging/](./packaging/)

Producing distributable artifacts (zip, deb, nsis, …).

- xmake-xpack

### [testing/](./testing/)

Tests for your project and for xmake itself.

- xmake-tests, xmake-unit-tests

### [performance/](./performance/)

Build-speed knobs — caches, distributed compilation, remote builds, optimization playbook.

- xmake-build-optimization, xmake-build-cache
- xmake-distributed-compilation, xmake-remote-compilation

### [scripting/](./scripting/)

Writing Lua inside `xmake.lua` — scripts, modules, plugins/tasks, async jobs, graphs, color output.

- xmake-scripting, xmake-script-modules
- xmake-plugins, xmake-custom-plugins
- xmake-async-jobs, xmake-graph-module, xmake-color-output

### [ops/](./ops/)

Running and debugging xmake itself — env vars, themes, troubleshooting, dev builds.

- xmake-env-vars, xmake-theme, xmake-troubleshooting, xmake-dev

### [languages/](./languages/)

Building non-C/C++ languages — per-language skills with target kinds, toolchains, interop.

- xmake-rust, xmake-go, xmake-swift, xmake-objc
- xmake-dlang, xmake-fortran, xmake-cuda, xmake-zig
- xmake-nim, xmake-pascal, xmake-vala, xmake-csharp, xmake-kotlin

## Authoring notes

- Keep each `SKILL.md` focused — the `description` field is what the agent reads to decide whether to load the skill, so make it specific about *when* to use it.
- Put long reference material under `references/` so `SKILL.md` stays small.
- Ground every example in real, verified Xmake behavior — no invented APIs.
- Cross-link between skills with a "When to branch out" section at the bottom.
- When adding a new skill, drop it in the most natural category and add a line to that category's `README.md`. If it doesn't fit anywhere, consider whether a new category is justified — aim to keep the top-level flat.

## Total

Roughly 50 skills across 11 categories.
