# scripting

Writing Lua inside `xmake.lua` — the script domain, `import()`, reusable modules, native (C/C++) Lua modules, custom plugins and tasks, async/parallel jobs, graphs, and colored output.

| Skill | Topic |
| --- | --- |
| [xmake-scripting](./xmake-scripting/) | Script domain, `import`, module convention, native modules (C shared/binary) |
| [xmake-script-modules](./xmake-script-modules/) | Built-in modules: `os`/`io`/`path`/`table`/`hash`/`try`-`catch` |
| [xmake-plugins](./xmake-plugins/) | Using built-in plugins — project generator, doxygen, compile_commands |
| [xmake-custom-plugins](./xmake-custom-plugins/) | Authoring `task(...)` / plugins — `set_menu`, distribution, importing core |
| [xmake-async-jobs](./xmake-async-jobs/) | `async.runjobs`, `async.jobgraph`, scheduler, semaphores, rule parallelism |
| [xmake-graph-module](./xmake-graph-module/) | `core.base.graph` — generic DAG data structure, topo sort, cycle detection |
| [xmake-color-output](./xmake-color-output/) | `cprint` + `${color}` tags, emoji, truecolor, disabling |
