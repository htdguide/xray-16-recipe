# src/xrCore/Threading/TaskManager.hpp

> Declares the work-stealing scheduler: the worker registry, the create/push/run surface, and the process-wide instance.

**Needs** — [`TaskManager.cpp`](TaskManager.cpp.md) · [`Task.hpp`](Task.hpp.md) · [`Event.hpp`](Event.hpp.md)
**Used by** — [`DetailManager.cpp`](../../Layers/xrRender/DetailManager.cpp.md) · [`ParallelFor.hpp`](ParallelFor.hpp.md) · [`ParallelForEach.hpp`](ParallelForEach.hpp.md) · [`Task.hpp`](Task.hpp.md) · [`TaskManager.cpp`](TaskManager.cpp.md) · [`xrCore.cpp`](../xrCore.cpp.md) · [`xr_object_list.cpp`](../../xrEngine/xr_object_list.cpp.md) · [`SoundRender_Emitter.cpp`](../../xrSound/SoundRender_Emitter.cpp.md)
**Tier floor** — T2: threads, atomics and a lock-free-ish queue. It is T2 rather than T1 only because nothing here is a byte layout — [`Task.hpp`](Task.hpp.md) carries that part.

## Purpose

Declares the surface implemented in [`TaskManager.cpp`](TaskManager.cpp.md). One instance exists per process, reachable as a global, and every parallel operation in the engine goes through it.

## Exported units

- **`TaskManager`** — owns the worker registry, the worker threads and the run/pause/stop state. Constructing it registers the constructing thread as worker zero; destroying it drains outstanding work and joins every thread.
- **`TaskManager.SpawnThreads`** — start the worker threads. Separate from construction because the engine registers a second thread of its own first.
- **`TaskManager.RegisterThisThreadAsWorker`** / **`UnregisterThisThreadAsWorker`** — enrol or withdraw the calling thread. A registered thread has its own queue and can be stolen from.
- **`TaskManager.CreateTask`** — build a task, optionally parented to another, without running it.
- **`TaskManager.PushTask`** — enqueue a task for any worker to run.
- **`TaskManager.RunTask`** — run a task inline on the calling thread.
- **`TaskManager.AddTask`** — create and push in one step; the form nearly every caller uses.
- **`TaskManager.Wait`** — run other tasks until the given task's subtree is finished. A flag makes the main thread pump the windowing layer's event queue while it waits.
- **`TaskManager.ExecuteOneTask`** — take one task from anywhere and run it; reports whether it found one.
- **`TaskManager.Pause`** — stop workers from taking new work without stopping them.
- **`TaskManager.GetWorkersCount`**, **`GetCurrentWorkerID`**, **`GetStats`** — the registry size, this thread's worker index, and the cumulative allocated/pushed/finished counters.
- **`TaskScheduler`** — the process-wide instance. It may be absent: code that runs before it is created must say so explicitly rather than wait on it.
