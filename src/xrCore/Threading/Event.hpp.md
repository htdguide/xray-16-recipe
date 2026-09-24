# src/xrCore/Threading/Event.hpp

> Declares the one-shot wake-up: a thread waits, another signals, exactly one waiter proceeds.

**Needs** — [`Event.cpp`](Event.cpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Event.cpp`](Event.cpp.md) · [`TaskManager.cpp`](TaskManager.cpp.md) · [`TaskManager.hpp`](TaskManager.hpp.md)
**Tier floor** — T2: a condition variable with a sticky flag.

## Purpose

Declares the surface implemented in [`Event.cpp`](Event.cpp.md). This is the primitive the task scheduler's workers sleep on: a worker with nothing to do waits, a thread that pushes work signals, and one sleeper wakes.

## Exported units

- **`Event`** — constructible as a fresh unsignalled event, as an empty handle, or as a wrapper around a handle the platform already gave out (the engine adopts a few of these from the windowing layer).
- **`Event.Set`** — move to the signalled state, releasing one waiter.
- **`Event.Reset`** — move back to the unsignalled state explicitly.
- **`Event.Wait`** — block until signalled, consuming the signal. An overload takes a millisecond deadline and reports whether the signal arrived in time.
- **`Event.GetHandle`**, **`Event.Valid`** — the adopted handle and whether one is held; used only where an engine event must be handed back to the platform.
