# src/xrCore/Threading/ParallelFor.hpp

> Turns a range into a binary tree of tasks: a range that is still bigger than its grain splits in half and hands one half to a new task, until every leaf is small enough to run.

**Needs** — [`TaskManager.hpp`](TaskManager.hpp.md) · [`Task.hpp`](Task.hpp.md)
**Used by** — [`HOM.cpp`](../../Layers/xrRender/HOM.cpp.md) · [`SkeletonXSkinXW_CPP.cpp`](../../Layers/xrRender/SkeletonXSkinXW_CPP.cpp.md) · [`ParallelForEach.hpp`](ParallelForEach.hpp.md) · [`cover_manager.cpp`](../../xrGame/cover_manager.cpp.md)
**Tier floor** — T2: recursive range decomposition over a task scheduler.

## Purpose

The scheduler in [`TaskManager.cpp`](TaskManager.cpp.md) deals in individual tasks. Almost every caller instead has "do this to each of N things". This file is the bridge, and the decision it encodes is **recursive halving rather than up-front partitioning**: the range is not cut into one piece per worker at the start, it is split lazily, so a half that turns out to be slow gets split again by whichever worker picks it up. That is what keeps the work balanced when the per-element cost varies, which in this engine it always does.

## State

```text
RECORD Range<T>                  # T is a position: an integer, or a sequence cursor
  first : T
  last  : T                      # half-open: [first, last)
  grain : int                    # invariant: >= 1; a range at or below it does not split
```

**Invariant** — the default grain is the range's length divided by the number of workers, floored, and raised to one if that is zero. So a range constructed without an explicit grain produces roughly one leaf per worker — and *then* those leaves may still be stolen and run out of order, which is where the balancing comes from. A caller who knows the per-element cost is uneven passes a smaller grain and buys finer balancing at the price of more tasks.

**Invariant** — splitting mutates the *original* range to become the second half and returns a new range holding the first half. The two halves are contiguous and together cover the original exactly; the split point is the midpoint by count, so an odd range leaves the extra element in the second half.

## `Range`

**Contract** — constructible from two positions, optionally with an explicit grain. Exposes the range as a sequence so a leaf body can simply iterate it. Reports its length, whether it is empty, and whether it is still splittable (its grain is strictly less than its length).

**Notes** — the length is computed by subtraction for numeric positions and by distance for cursors, chosen at build time. That distinction is incidental; what matters is that the length must be cheap, because it is asked on every split decision.

A grain of zero is rejected: a range that never stops splitting recurses until the task ring overflows.

## `xr_parallel_for`

**Contract** — runs a body over a range, in parallel, and returns the root task. Four forms exist, crossing two choices: whether the caller blocks until the whole range is done, and whether the work is parented to an existing task (so that an outer wait covers it). The body receives a *range*, not an element — it is expected to loop.

Blocking without a scheduler present is a checked error: the caller must state explicitly that it knows the scheduler does not exist yet by choosing the non-blocking form. That is deliberate, because the engine runs some of this before the scheduler is up and a silent synchronous fallback would hide it.

```text
FUNCTION parallel_for(range, body, parent: optional<Task>, wait: bool) -> Task
  root <- scheduler.add_task(parent, Splitter{ range, body })
  IF wait
    require the scheduler exists
    scheduler.wait(root)
  RETURN root

# the work each task runs:
FUNCTION Splitter(task, range, body)
  IF range.is_splittable()
    left <- split off the first half of range   # range becomes the second half
    scheduler.add_task(task, Splitter{ left,  body })
    scheduler.add_task(task, Splitter{ range, body })   # the second half, as a new task
  ELSE
    body(range)
```

**Invariants** — both halves become *children of the currently running task*, so the completion count in [`Task.hpp`](Task.hpp.md) makes the root's finish mean "every leaf has run". The splitting task itself does no work after spawning; it finishes immediately, and its parent's count is held up by its two children.

**Notes** — the splitting task pushes both halves rather than running one inline. Running one half inline would save a task and keep the data in this core's cache, and is the usual optimization; the original does not do it, and a rebuild that adds it changes the execution order (and therefore the steal pattern) without changing the result.

The splitter re-pushes *itself* — the same closure, with the range now narrowed to the second half — as the second child. That is why the closure must be copyable and why it must fit the task's inline byte area: a range is two positions and an integer, a body is whatever the caller captured, and the sum of the two is capped by [`Task.hpp`](Task.hpp.md)'s storage limit. A caller whose body captures too much gets a build-time failure, not a runtime one.

The body is held by reference inside the splitter in the helper that wraps element-wise iteration ([`ParallelForEach.hpp`](ParallelForEach.hpp.md)), which means the caller's body must outlive the whole parallel region. With the blocking form that is automatic; with the non-blocking form it is the caller's obligation and is not checked.
