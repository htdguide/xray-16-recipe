# src/xrCore/Threading — the work-stealing scheduler and the primitives under it

Part of chapter 6, [`src/xrCore`](../README.md). Thirteen files. The scheduler in the middle
of them is the single largest piece of machinery in this chapter that is not the filesystem.

## What this module is responsible for

Three layers, and it is worth separating them before reading any twin.

**The primitives** — a recursive mutex, a scope guard for it, and a one-shot wake-up event.
These are thin, and in a rebuild whose language ships its own they mostly vanish. What does
not vanish is the one non-standard thing the mutex carries: a hold counter readable *without*
taking the lock, so that "is anyone in here" can be asked cheaply.

**The scheduler** — a work-stealing task system. Each worker thread owns a ring of pending
tasks and a ring of task storage; it takes from its own ring first, steals from a random
peer next, and sleeps on one shared event when there is nothing anywhere. This is what every
parallel loop in the engine runs on.

**The parallel-loop shorthands** — a range that splits itself into a binary tree of tasks
until each leaf is below a grain size, and an element-wise form of the same.

Plus one thing that does not fit the layering and is easy to miss: **every engine thread must
put its own floating-point unit into flush-to-zero mode before running anything**. That rule
lives in [`ThreadUtil.h`](ThreadUtil.h.md) and a thread that skips it produces different
numbers from its peers.

## Where it sits

It rests on [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
and on the allocator. Everything from chapter 7 onward consumes it: the collision database's
concurrent queries, the renderer's visibility pass, level loading, and the audio streaming
thread.

## The load-bearing ideas

**A task is exactly one cache line.** The call, the parent link, the outstanding-job count
and a fixed inline byte area that holds the closure *and, afterwards, its result*, all sized
to fit one line. That is the constraint the whole scheduler is built around, and it is why
the closure has a size limit rather than being allocated. [`Task.hpp`](Task.hpp.md) states
the limit; exceeding it is a build failure, deliberately.

**Take from yourself, steal from a random peer, then sleep.** The three-step rule is the
whole scheduling policy. Random victim selection, rather than round-robin, is what avoids the
convoy where every idle worker probes the same busy one.

**Sleeping is on one shared event, not per worker.** A worker with nothing to do waits on the
same object every other idle worker waits on, and any push wakes them. That trades a thundering
herd on a single task for a much simpler wake path, and the twin says where the trade shows.

**Parallel loops are trees, not partitions.** A range bigger than its grain splits in half and
hands one half to a new task, recursively. Nothing decides up front how many pieces there
will be, so a loop whose work is unevenly distributed still balances. The grain size is the
only tuning knob and it is the caller's.

**The parent's outstanding count is how a task knows its children are done.** There is no join
object; completion propagates up through counts. A rebuild with structured concurrency
expresses this directly and should.

**A named thread is a debuggable thread.** Naming exists purely so a stack trace and a
profiler show something other than a number, and it is worth the platform-specific code it
costs.

## The twins

| File | Role |
|---|---|
| [`Task.hpp`](Task.hpp.md) | **One unit of parallel work**, sized to exactly one cache line: call, parent, outstanding count, and an inline area for the closure and then its result. Substantive. |
| [`TaskManager.hpp`](TaskManager.hpp.md) | Declares the scheduler: worker registry, create/push/run surface, and the process-wide instance. |
| [`TaskManager.cpp`](TaskManager.cpp.md) | **The work-stealing scheduler**: own ring first, random peer next, one shared sleep event. The largest page here. Substantive. |
| [`ParallelFor.hpp`](ParallelFor.hpp.md) | **Range splitting**: a range above its grain halves itself into two tasks, recursively, until every leaf fits. Substantive. |
| [`ParallelForEach.hpp`](ParallelForEach.hpp.md) | The element-wise shorthand over the same splitting. |
| [`Lock.hpp`](Lock.hpp.md) | Declares the engine's mutex: recursive, countable, optionally instrumented. |
| [`Lock.cpp`](Lock.cpp.md) | **The recursive mutex** and the hold counter beside it that answers "is anyone in here" without acquiring. |
| [`ScopeLock.hpp`](ScopeLock.hpp.md) | Declares the guard that holds a mutex for the length of a block. |
| [`ScopeLock.cpp`](ScopeLock.cpp.md) | Enter on the way in, leave on every way out — including the failure path. |
| [`Event.hpp`](Event.hpp.md) | Declares the one-shot wake-up: one waiter proceeds per signal. |
| [`Event.cpp`](Event.cpp.md) | One thread sleeps until another says go, with the signal consumed by the sleeper. |
| [`ThreadUtil.h`](ThreadUtil.h.md) | Names, priorities, and **the rule every engine thread must obey before running anything**: set this processor's floating-point mode. Substantive. |
| [`ThreadUtil.cpp`](ThreadUtil.cpp.md) | Giving a thread a name a debugger shows, and mapping the engine's priority names onto the host's. |
