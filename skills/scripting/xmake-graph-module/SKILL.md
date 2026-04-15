---
name: xmake-graph-module
description: Use when writing xmake scripts that need a generic graph / DAG data structure — `core.base.graph` for directed or undirected graphs, adding vertices/edges, topological sort, cycle detection, cloning, reversing. Lower-level than `async.jobgraph` — use this when you need the graph but not the scheduler.
---

# `core.base.graph` — Graph Data Structure

`core.base.graph` is xmake's general-purpose graph module. It implements both directed and undirected graphs with topological sort and cycle detection. Use it when you need to model dependencies / relationships in a custom script but don't want to run them as async jobs (for that, use `async.jobgraph`, which wraps this).

Must be imported:

```lua
import("core.base.graph")
```

## 1. Create a graph

```lua
local dag = graph.new(true)    -- directed (DAG)
local ug  = graph.new(false)   -- undirected
```

## 2. Vertices and edges

```lua
g:add_vertex("a")
g:add_vertex("b")
g:add_vertex("c")

g:add_edge("a", "b")           -- a → b  (directed)
g:add_edge("b", "c")

print(g:has_vertex("a"))       -- true
print(g:has_edge("a", "b"))    -- true
print(#g:vertices())           -- 3
```

Vertices can be any value (strings, numbers, tables — anything you can use as a table key).

### Shorthand

`add_edge` auto-adds endpoints:

```lua
local g = graph.new(true)
g:add_edge("a", "b")           -- "a" and "b" added automatically
g:add_edge("b", "c")
```

### Inspect

```lua
for _, v in ipairs(g:vertices()) do print(v) end
for _, e in ipairs(g:edges()) do
    print(e:from(), "->", e:to())
end
for _, e in ipairs(g:adjacent_edges("a")) do
    print(e:to())              -- outgoing edges from "a"
end
```

### Remove

```lua
g:remove_vertex("a")           -- also removes incident edges
g:clear()                      -- reset
print(g:empty())               -- true
```

## 3. Topological sort (DAG only)

```lua
local g = graph.new(true)
g:add_edge("parse",     "analyze")
g:add_edge("analyze",   "optimize")
g:add_edge("optimize",  "emit")

local order, has_cycle = g:topo_sort()
if has_cycle then
    raise("cycle detected!")
end
for _, v in ipairs(order) do
    print(v)     -- parse, analyze, optimize, emit
end
```

`topo_sort()` returns `(list, has_cycle)`. If there's a cycle, use `find_cycle()` to get the offending path:

```lua
local cycle = g:find_cycle()
if cycle then
    raise("cycle: %s", table.concat(cycle, " -> "))
end
```

## 4. Partial (incremental) topo sort

For work loops where you want to process vertices as their dependencies become ready:

```lua
g:partial_topo_sort_reset()
while true do
    local v, has_cycle = g:partial_topo_sort_next()
    if has_cycle then raise("cycle") end
    if not v then break end

    process(v)
    g:partial_topo_sort_remove(v)    -- mark done, unblock dependents
end
```

Use this when processing is interleaved with new information (e.g. parallel workers feeding back completion signals).

## 5. Other utilities

```lua
local copy = g:clone()              -- deep copy
local rev  = g:reverse()            -- reverse all edges (directed only)
g:dump()                            -- debug dump to stdout
print(g:is_directed())
```

## 6. Typical uses in xmake scripts

### Custom build pipeline in a task

```lua
task("pipeline")
    on_run(function ()
        import("core.base.graph")

        local g = graph.new(true)
        for _, step in ipairs(pipeline_steps) do
            g:add_vertex(step.name)
            for _, dep in ipairs(step.deps or {}) do
                g:add_edge(dep, step.name)
            end
        end

        local order, cycle = g:topo_sort()
        if cycle then
            raise("pipeline has a cycle")
        end
        for _, name in ipairs(order) do
            run_step(name)
        end
    end)
```

### Dependency analysis of target metadata

```lua
import("core.base.graph")
import("core.project.project")

local g = graph.new(true)
for name, target in pairs(project.targets()) do
    g:add_vertex(name)
    for _, dep in ipairs(target:get("deps") or {}) do
        g:add_edge(dep, name)
    end
end

local cycle = g:find_cycle()
if cycle then
    cprint("${red}cycle: %s", table.concat(cycle, " -> "))
end

-- print dependency-order build plan
for _, name in ipairs(g:topo_sort()) do
    print(name)
end
```

### Detecting stale edges / cycle in custom rule

```lua
rule("codegen")
    before_build(function (target)
        import("core.base.graph")
        local g = graph.new(true)
        -- add edges based on #include scan of generated files
        ...
        if g:find_cycle() then
            raise("include cycle in generated headers")
        end
    end)
```

## 7. `graph` vs `async.jobgraph`

| Feature | `core.base.graph` | `async.jobgraph` |
| --- | --- | --- |
| Data structure | generic DAG / undirected | DAG tailored to jobs |
| Vertex payload | any value | name + job function |
| Execution | none — you iterate yourself | scheduled by `async.runjobs` |
| Groups | not built in | yes |
| Cycle detection | `find_cycle()` / `topo_sort()` | yes |

Rule of thumb:

- Need a graph for **analysis / planning** → `core.base.graph`.
- Need a graph to **run jobs in parallel** → `async.jobgraph` (which uses `core.base.graph` internally).

## Pitfalls

- **`topo_sort()` on an undirected graph.** Doesn't make sense; xmake raises. Use `graph.new(true)` for topo sort.
- **Cycles through self-loops.** `g:add_edge("a", "a")` is a cycle. `find_cycle()` will flag it.
- **Vertices are keyed by identity.** For tables, two distinct tables with the same contents are different vertices. Use stable identifiers (strings) unless you want identity semantics.
- **Large graphs — no persistent storage.** The graph lives in memory only. For persistent DAG state across xmake runs, serialize yourself.
- **Edge weights.** Not supported — this is an unweighted graph. Roll your own weight table if you need Dijkstra/etc.

## When to branch out

- Running the DAG as parallel jobs → `xmake-async-jobs`
- Working with target dependency metadata → `xmake-targets`, `xmake-scripting`
- Writing rules that need per-file ordering → `xmake-rules` and `xmake-async-jobs`
