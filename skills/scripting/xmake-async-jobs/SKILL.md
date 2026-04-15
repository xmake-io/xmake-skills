---
name: xmake-async-jobs
description: Use when running parallel / async work inside xmake scripts — `async.runjobs` for parallel job execution, `async.jobgraph` (DAG) for dependency-ordered jobs, `async.jobpool` (tree structure), `core.base.scheduler` coroutine scheduling, semaphores, and `on_build_files(..., {jobgraph = true})` in rules. For running external compile jobs faster, see `xmake-build-optimization`.
---

# Async Jobs, Parallelism & Scheduling

Xmake's Lua layer has a cooperative coroutine scheduler underneath. Any `on_build`/`on_build_files`/`on_install` hook you write runs inside it, and can spawn parallel subjobs via `async.runjobs` (with three different input shapes) or by yielding explicitly via the scheduler.

Use this skill when writing a **rule** or **task** that needs to do work in parallel — file generation, batch downloads, distributed jobs, etc.

## 1. `async.runjobs` — the main entry point

```lua
import("async.runjobs")

runjobs("my-jobs", function (index, total, opt)
    print("job %d/%d", index, total)
    os.sleep(500)
end, {
    total = 100,
    comax = 6
})
```

Parameters:

- `name` — used in scheduler output / timing.
- `jobs` — can be a function (called `total` times), a `jobpool`, or a `jobgraph`.
- Options:
  - `total` — required when `jobs` is a function.
  - `comax` — max concurrent coroutines (default 4).
  - `timeout` — ms for `on_timer` callback (default 500).
  - `on_timer = function(indices)` — periodic callback with running job indices.
  - `on_exit = function(abort_errors)` — called after all jobs finish or error.
  - `waiting_indicator = true` — simple spinner; or `{chars = {'/', '-', '\\', '|'}}`.
  - `curdir` — set cwd for each job.
  - `isolate = true` — isolate coroutine environments.
  - `distcc = client` — farm out to a distcc client.
  - `remote_only = true` — force remote execution.

Example with timer:

```lua
runjobs("download", function (i, total, opt)
    os.vrunv("curl", {"-O", urls[i]})
end, {
    total = #urls,
    comax = 8,
    timeout = 1000,
    on_timer = function (indices)
        utils.vprint("still running: %s", table.concat(indices, ","))
    end
})
```

## 2. `async.jobgraph` — DAG-ordered jobs

When jobs have dependencies (A must finish before B), build a job graph:

```lua
import("async.jobgraph")
import("async.runjobs")

local jobs = jobgraph.new()

local function jobfunc(i, total, opt)
    print("job %s (%d/%d)", opt.progress:percent(), i, total)
end

jobs:add("root", jobfunc)
for i = 1, 3 do
    jobs:add("mid/" .. i, jobfunc)
    jobs:add_orders("mid/" .. i, "root")       -- mid/i depends on root
    for j = 1, 5 do
        local leaf = "leaf/" .. i .. "/" .. j
        jobs:add(leaf, jobfunc)
        jobs:add_orders(leaf, "mid/" .. i)
    end
end

runjobs("build", jobs, {comax = 4})
```

`jobgraph:add_orders(A, B, C, ...)` means **A depends on B, C, …** (all of them must finish first).

### Groups for batch dependency management

```lua
local jobs = jobgraph.new()

jobs:group("codegen", function ()
    for _, f in ipairs(proto_files) do
        jobs:add("codegen/" .. f, function() os.exec("protoc " .. f) end)
    end
end)

jobs:group("compile", function ()
    for _, f in ipairs(cpp_files) do
        jobs:add("compile/" .. f, function() compile(f) end)
    end
end)

-- all "compile/*" depend on all "codegen/*"
jobs:add_orders("compile", "codegen")
```

Single-call bulk dependency wiring — much cleaner than nested loops.

## 3. `async.jobpool` — hierarchical (tree) jobs

```lua
import("async.jobpool")
import("async.runjobs")

local jobs = jobpool.new()
local root = jobs:addjob("root", function(i, total, opt) print("root") end)

for i = 1, 3 do
    jobs:addjob("child/" .. i, function(i, total, opt)
        print("child")
    end, {rootjob = root})
end

runjobs("tree", jobs, {comax = 6})
```

Difference from jobgraph: jobpool is a tree (one parent per job), jobgraph is a DAG (multiple deps allowed). Use jobgraph unless you specifically want the simpler tree model.

## 4. Rules with parallel build-file hooks

Xmake rules can hand a jobgraph directly to the rule runner. This is how the built-in C/C++ rules drive parallel compilation.

```lua
rule("foo")
    on_build_files(function (target, jobgraph, sourcebatch, opt)
        local group_name = target:name() .. "/buildfiles"
        for _, sourcefile in ipairs(sourcebatch.sourcefiles) do
            local job_name = target:name() .. "/" .. sourcefile
            jobgraph:add(job_name, function(i, total, opt)
                -- compile the file
                os.vrunv("my-compiler", {"-c", sourcefile, "-o", job_name .. ".o"})
            end, {groups = group_name})
        end
        -- wire in inter-target deps
        jobgraph:add_orders(other_target:name() .. "/buildfiles", group_name)
    end, {jobgraph = true})
```

Key: `{jobgraph = true}` in the `on_build_files` options. Xmake then passes a shared jobgraph and schedules everything together — so your rule's work runs in parallel with the rest of the build, respecting dependencies.

Without `{jobgraph = true}`, `on_build_files` is a plain hook and you'd use `runjobs` yourself.

## 5. Explicit scheduler operations

```lua
import("core.base.scheduler")

-- suspend / resume
scheduler.co_suspend()
scheduler.co_resume(coroutine_object)

-- sleep (non-blocking for other coroutines)
scheduler.co_sleep(1000)          -- ms

-- yield to the scheduler
scheduler.co_yield()

-- query state
scheduler.co_running()            -- current coroutine name
```

Inside a `runjobs` callback you can `os.sleep(ms)` safely — it yields via the scheduler so other jobs keep running.

## 6. Semaphores

For rate-limiting / bounded-concurrency patterns beyond `comax`:

```lua
import("core.base.semaphore")

local sem = semaphore.new("net-sem", 4)   -- max 4 concurrent
runjobs("downloads", function (i, total, opt)
    sem:wait(-1)                           -- -1 = wait forever
    try { function ()
        os.vrunv("curl", {"-O", urls[i]})
    end, finally { function ()
        sem:post()
    end}}
end, {total = #urls, comax = 32})
```

- `semaphore.new(name, count)` — create.
- `sem:wait(timeout_ms)` — `-1` = forever, `0` = non-block, `>0` = bounded.
- `sem:post()` — release.

Use a semaphore when `comax` is too coarse (e.g. allow 32 total jobs but only 4 simultaneous network calls).

## 7. Distributed jobs

```lua
import("async.runjobs")
import("private.service.distcc_build.client", {alias = "distcc"})

runjobs("compile", jobs, {
    comax = 16,
    distcc = distcc.singleton()
})
```

See `xmake-distributed-compilation` for the full distcc setup.

## 8. Progress reporting

Inside a job function, `opt.progress` gives:

```lua
runjobs("build", function (i, total, opt)
    cprint("${bright}[%3d%%]${clear} building %d/%d", opt.progress:percent(), i, total)
end, {total = 200})
```

Methods: `progress:current()`, `progress:total()`, `progress:percent()`.

## 9. Typical patterns

### Parallel codegen in a rule

```lua
rule("protobuf")
    set_extensions(".proto")
    on_build_files(function (target, jobgraph, sourcebatch, opt)
        for _, sourcefile in ipairs(sourcebatch.sourcefiles) do
            jobgraph:add("protoc/" .. sourcefile, function (i, total, opt)
                os.vrunv("protoc", {"--cpp_out=gen", sourcefile})
            end, {groups = target:name() .. "/protoc"})
        end
    end, {jobgraph = true})
```

### Batch downloads with rate limiting

```lua
import("async.runjobs")
import("core.base.semaphore")

local sem = semaphore.new("dl", 4)
runjobs("download", function (i, total, opt)
    sem:wait(-1)
    try { function ()
        os.vrunv("curl", {"-sS", "-O", urls[i]})
    end, finally { function ()
        sem:post()
    end}}
end, {total = #urls, comax = 32})
```

### Split a heavy task into a jobgraph

```lua
local jobs = jobgraph.new()
jobs:group("parse",     function () for _, f in ipairs(files) do jobs:add("parse/"    .. f, ...) end end)
jobs:group("transform", function () for _, f in ipairs(files) do jobs:add("transform/" .. f, ...) end end)
jobs:group("emit",      function () for _, f in ipairs(files) do jobs:add("emit/"     .. f, ...) end end)

jobs:add_orders("transform", "parse")
jobs:add_orders("emit",      "transform")

runjobs("pipeline", jobs, {comax = 8})
```

## Pitfalls

- **Blocking calls starve the scheduler.** `os.execv` is fine (spawns a process, yields while waiting); a tight CPU loop blocks everyone. Break long computations with `scheduler.co_yield()` if really needed.
- **Shared mutable state across jobs.** No locking primitives — if jobs write to the same Lua table, races. Use `isolate = true` or keep writes to a final synchronization step.
- **Jobgraph cycles.** Deadlock. The scheduler detects them and raises at setup time, but check before adding edges if you're building dynamically.
- **`comax` too low for I/O workloads.** Default 4 is fine for CPU; bump to 16–64 for network-bound jobs.
- **Forgetting `{jobgraph = true}` in a rule.** Without it, the callback receives `sourcebatch` positionally differently — the signature changes. Match the signature you expect.
- **Mixing `runjobs` inside a `runjobs` job.** Legal but easy to nest concurrency wrong. Usually cleaner to flatten into one jobgraph.

## When to branch out

- Writing rules / `on_build_files` / `set_extensions` → `xmake-rules`
- Parallelism at the *whole build* level (`-jN`, distcc, cache) → `xmake-build-optimization`
- Distributed compile service → `xmake-distributed-compilation`
- The `core.base.graph` DAG module (lower-level than jobgraph) → `xmake-graph-module`
