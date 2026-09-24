# src/utils/mp_gpprof_server/sake_worker.h

> Declares the single thread that owns the remote session, and the task queue that is the only way to reach it.

**Needs** — [`gamespy_sake.h`](gamespy_sake.h.md) · [`threads.h`](threads.h.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`requests_processor.cpp`](requests_processor.cpp.md) · [`requests_processor.h`](requests_processor.h.md) · [`sake_worker.cpp`](sake_worker.cpp.md)

**Tier floor** — T2: a worker thread and a queue; the session it owns is T1 only because the vendor library says so.

## Purpose

Declares the surface implemented in [`sake_worker.cpp`](sake_worker.cpp.md). The whole
type exists to enforce one rule: **the remote session is not thread-safe and must be
touched from exactly one thread.** Rather than lock it, the tool gives it a thread of its
own and makes every other thread submit a task.

The queue holds callable tasks that are handed the session when they run, and a task may
ask to be run again — which is how the tool's single long-running request pump is
expressed as a task that never finishes.

## Exported units

- `sake_worker` — the owning thread and its queue. Constructing it blocks until the
  session is up, and fails hard if it is not.
- `add_task` — submit a task. Safe from any thread.
- `sake_task_proc_t` — a task: takes its own context and the session, and returns whether
  it wants to run again.

**Notes**

- The header carries an explicit warning not to reorder the record's fields. That is a
  real constraint: the worker thread starts during construction and immediately touches the
  lock, the condition and the queue, so all three must be fully constructed before the
  thread handle is. A rebuild expresses this by starting the thread *after* construction
  rather than by relying on field order.
