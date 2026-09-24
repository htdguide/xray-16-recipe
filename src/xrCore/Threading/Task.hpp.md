# src/xrCore/Threading/Task.hpp

> One unit of parallel work: a call, a parent, an outstanding-job count, and a fixed inline byte area that holds both the closure and, afterwards, its result — all sized to exactly one cache line.

**Needs** — [`TaskManager.hpp`](TaskManager.hpp.md) · [`xrDebug.h`](../xrDebug.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`ParallelFor.hpp`](ParallelFor.hpp.md) · [`TaskManager.cpp`](TaskManager.cpp.md) · [`TaskManager.hpp`](TaskManager.hpp.md)
**Tier floor** — T1: the type's size is pinned to the hardware's cache line, its fields are ordered to fit, and the closure is stored inline in raw bytes rather than behind an indirection. Every one of those is a layout decision.

## Purpose

The scheduler's currency. A task is deliberately a *fixed-size, self-contained* record with no allocation of its own: the work to run is copied into a byte area inside the task, the result — if any — is written back into that same area when the work completes, and the task's storage comes from a per-thread ring, never from the heap. That combination is what makes pushing a task cost a few stores instead of an allocation.

A task also carries the *tree* structure the scheduler uses to answer "is this whole sub-computation finished": each task counts its own outstanding jobs, a child raises its parent's count when created, and a completing task walks up the chain decrementing until it meets an ancestor that still has other work outstanding.

## State

```text
RECORD Task                        # invariant: sizeof(Task) == TASK_SIZE, exactly
  user_data : bytes[TASK_SIZE - sizeof(Data)]
  data      : Data

RECORD Data                        # ordered largest field first, to pack without gaps
  call       : function(Task) -> nothing
  parent     : optional<Task>
  jobs       : int (16-bit), atomic   # invariant: >= 1 while unfinished; 1 counts
                                      #   the task itself, each live child adds one
  has_result : bool, atomic
```

**`TASK_SIZE`** is 64 bytes on 32-bit architectures and 128 on 64-bit ones, raised to the hardware's destructive-interference size where the language reports one truthfully. The number is not about how much a closure needs; it is about **false sharing**. Tasks live adjacent in a per-thread array and are written by different threads as they are stolen, so two tasks sharing a cache line would ping-pong that line between cores. Pinning the size to a whole line (or a pair of lines, which is what 128 is on a machine whose prefetcher pulls pairs) removes that entirely. A rebuild must make the same decision explicitly — the number is a property of the target machine, not of the code.

The user area is therefore `TASK_SIZE` minus the header, about 40 bytes on a 64-bit build. **A closure larger than that is rejected at build time**, and so is a result larger than it. That is a hard constraint on every caller: work that needs more context captures a pointer to it rather than the thing itself, and owns that thing's lifetime for the duration.

**Invariant** — `jobs` reaching zero means this task *and its whole subtree* are done. It starts at one (the task itself), a child creation raises it, and the task's own completion plus each child's completion lowers it. A task that has already finished may not take a child; doing so is a checked error, because the parent's count would never be observed again.

**Invariant** — the user area holds the closure *before* the call and the result *after*, in the same bytes. The closure is destroyed before the result is placed there. `has_result` is published with release ordering and read with a matching ordering, so a thread that sees the flag also sees the bytes.

## `Task`

**Contract** — a task is constructed in place inside storage the scheduler hands out, never allocated. Construction copies the work into the user area (an empty, trivially-copyable closure is not copied at all — there is nothing to copy) and, if a parent was given, raises the parent's outstanding count. Tasks cannot be copied or moved: their address is their identity, and the scheduler's queues hold pointers to them.

**Contract (`GetData`)** — returns the result bytes, or nothing if the work has not finished or produced none. The caller must know the result's type; nothing records it.

**Contract (`IsFinished`, `GetJobsCount`)** — an observation of the outstanding count, safe to read from any thread with no ordering guarantee attached. The scheduler's wait loop spins on it.

**Contract (`AvailableDataStorageSize`)** — how many bytes a closure or a result may occupy. Callers use it to decide whether to capture by value or by reference.

## Running a task

**Contract** — invoking a task runs its stored work, destroys the closure, optionally stores the result, and then settles the completion counts up the ancestor chain. Called only by the scheduler, exactly once per task; running one twice is a checked error.

```text
FUNCTION run(t: Task)
  t.call(t)                        # dispatches to the stored work; see below

  node <- t
  LOOP                             # settle completion upward
    remaining <- ATOMICALLY decrement node.jobs, taking the new value
    IF remaining > 0
      BREAK                        # this subtree still has outstanding work
    IF node.parent is none
      BREAK                        # reached the root
    node <- node.parent
```

**Invariants** — the decrement is the point at which a waiter may observe completion, so everything the work published must be visible before it. It is performed with acquire-release ordering for exactly that reason.

The walk stops at the *first* ancestor that still has outstanding jobs, which is correct: that ancestor will itself walk further up when its own last child finishes. Walking unconditionally to the root would decrement ancestors that are not yet done.

## The four shapes of work

The stored work is invoked through a function chosen at build time from four cases, by whether the work takes the task as an argument and whether it returns anything:

| Takes the task | Returns | What happens |
|---|---|---|
| no | nothing | call it; destroy the closure |
| yes | nothing | call it with the task; destroy the closure |
| no | a value | call it; destroy the closure; move the value into the user area; publish the result flag |
| yes | a value | same, with the task passed in |

**Notes** — passing the task *to* the work is what lets work spawn children: a parallel loop's body splits its range and creates two child tasks parented to the task it was handed. That is the whole recursive-decomposition mechanism in [`ParallelFor.hpp`](ParallelFor.hpp.md).

Destroying the closure before writing the result into the same bytes is load-bearing and easy to get wrong: the result is moved to a temporary first, then the closure is destroyed, then the temporary is moved into the area. Doing it in any other order either leaks the closure or overwrites it while it is still live.

A closure that needs no destruction is not destroyed, which is a cost saving on the common case of a small capture-by-value lambda; a rebuild in a language with automatic reclamation ignores this whole paragraph.

## The rebuild's real constraints

Strip the language and three requirements remain:

1. A task is a **fixed-size** record occupying a whole cache line or a whole pair of them, because tasks are written by different cores from adjacent storage.
2. Its work and its result share **inline** storage with a hard size cap, because the scheduler must not allocate.
3. Completion is a **counted tree**, settled by one atomic decrement per task and a walk that stops at the first still-busy ancestor.
