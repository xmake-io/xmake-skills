# Skills

Each subdirectory is a self-contained [Agent Skill](https://www.anthropic.com/news/agent-skills) for a specific area of Xmake. Skills are organized by topic so they can be maintained and loaded independently — an agent only pulls in the skills relevant to the task at hand.

## Layout

```
skills/
  <skill-name>/
    SKILL.md           # required: frontmatter + instructions
    references/        # optional: supporting docs (loaded on demand)
    scripts/           # optional: helper scripts
```

## Index

| Skill | Topic |
| --- | --- |
| [xmake-basics](./xmake-basics/) | Installing Xmake, creating a project, `xmake.lua` basics |
| [xmake-targets](./xmake-targets/) | Target configuration: `target()`, `add_files`, `kind`, deps |
| [xmake-options](./xmake-options/) | User-defined options and build configuration |
| [xmake-packages](./xmake-packages/) | Package integration via `add_requires` / `xrepo` |
| [xmake-rules](./xmake-rules/) | Built-in and custom rules |
| [xmake-toolchains](./xmake-toolchains/) | Toolchains and cross-compilation |
| [xmake-commands](./xmake-commands/) | CLI: `build`, `run`, `install`, `pack`, etc. |
| [xmake-tests](./xmake-tests/) | Writing and running tests |
| [xmake-plugins](./xmake-plugins/) | Plugins and custom tasks |

## Authoring notes

* Keep each `SKILL.md` focused — the `description` field is what the agent reads to decide whether to load the skill, so make it specific about *when* to use it.
* Put long reference material under `references/` so the main `SKILL.md` stays small.
* Ground every example in real, verified Xmake behavior — no invented APIs.
