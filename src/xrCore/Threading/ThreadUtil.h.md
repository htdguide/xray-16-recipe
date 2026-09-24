# src/xrCore/Threading/ThreadUtil.h

> Names, priorities, and the one rule every engine thread must obey before it runs anything: initialize this processor's floating-point mode.

**Needs** — [`ThreadUtil.cpp`](ThreadUtil.cpp.md) · [`_math.h`](../_math.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`TaskManager.cpp`](TaskManager.cpp.md) · [`ThreadUtil.cpp`](ThreadUtil.cpp.md) · [`xrCore.h`](../xrCore.h.md)
**Tier floor** — T2: thread creation and naming, plus one per-thread processor state that is itself T1 and lives in [`_math.cpp`](../_math.cpp.md).

## Purpose

Declares the priority vocabulary and the platform calls implemented in [`ThreadUtil.cpp`](ThreadUtil.cpp.md), and carries inline the two thread-launching helpers — which are here, rather than next door, because they are where the *engine's* thread-start protocol is enforced.

## Exported units

- **`priority_level`** — seven per-thread priorities from idle to time-critical.
- **`priority_class`** — six process-wide priorities from idle to real-time.
- **`GetCurrentThreadPriorityLevel`**, **`GetCurrentProcessPriorityClass`**, **`SetCurrentThreadPriorityLevel`**, **`SetCurrentProcessPriorityClass`** — read and write them; see [`ThreadUtil.cpp`](ThreadUtil.cpp.md).
- **`SetCurrentThreadName`** — give the calling thread a name visible to a debugger and to the profiler.
- **`RunThread`** — start a thread that runs a given callable with given arguments, returning a handle to join.
- **`SpawnThread`** — the same, detached.

## `RunThread` / `SpawnThread`

**Contract** — start a thread on a callable with its arguments, naming it before the callable runs. Returns a joinable handle, or detaches it. The arguments are moved into the new thread, so the caller's copies must not be relied on afterwards.

**Invariants** — every thread the engine starts does two things before touching the callable, and **both are required**:

```text
FUNCTION thread_entry(name, work, args)
  set_current_thread_name(name)     # so a debugger and the profiler can identify it
  initialize_cpu_thread()           # see _math.cpp: installs the crash handler for
                                    # this thread and sets the processor's
                                    # flush-denormals-to-zero mode
  work(args)
```

The second call is the load-bearing one. The floating-point unit's denormal handling is **per thread**, not per process: a worker started without it computes denormals the slow way, which shows up as a worker that is inexplicably several times slower than the others on the same data, and — worse — as results that differ in the last bits between the main thread and a worker. A rebuild whose threads are started by a library must find the equivalent hook, or the physics determinism criterion in §6 of the system requirements will fail intermittently.

**Notes** — the name is captured by value into the new thread, so it must be a string literal or otherwise outlive the launch. Every call site passes a literal.
