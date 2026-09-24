# src/xrCore/Threading/Lock.hpp

> Declares the engine's mutex: recursive, countable, and optionally instrumented.

**Needs** — [`Lock.cpp`](Lock.cpp.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`ppmd_compressor.cpp`](../Compression/ppmd_compressor.cpp.md) · [`StackTrace.cpp`](../Debug/StackTrace.cpp.md) · [`Notifier.h`](../Events/Notifier.h.md) · [`LocatorAPI.cpp`](../LocatorAPI.cpp.md) · [`LocatorAPI_auth.cpp`](../LocatorAPI_auth.cpp.md) · [`Lock.cpp`](Lock.cpp.md) · [`ScopeLock.cpp`](ScopeLock.cpp.md) · [`ScopeLock.hpp`](ScopeLock.hpp.md) · [`log.cpp`](../log.cpp.md) · [`xrDebug.h`](../xrDebug.h.md) · [`xrsharedmem.cpp`](../xrsharedmem.cpp.md) · [`xrstring.cpp`](../xrstring.cpp.md) · [`EventAPI.h`](../../xrEngine/EventAPI.h.md) · _and 7 more_
**Tier floor** — T2: a recursive mutex plus an atomic counter.

## Purpose

Declares the surface implemented in [`Lock.cpp`](Lock.cpp.md). One mutex type is used everywhere in the engine, and it is *recursive* — the same thread may enter it more than once — because several subsystems call back into themselves through code that re-locks.

## Exported units

- **`Lock`** — the mutex. Not copyable. Movable, which is unusual for a mutex and is explained in the implementation.
- **`Lock.Enter`** — block until held.
- **`Lock.TryEnter`** — take it if free, report whether it was taken, never block.
- **`Lock.Leave`** — release one level.
- **`Lock.IsLocked`** — whether the hold count is non-zero. An observation, not a guarantee: it may change the instant after it is read.
- **`set_add_profile_portion`** — in an instrumented build only, installs the callback that receives how long each acquisition blocked, keyed by the mutex's construction-time name. Compiled out otherwise, along with the name parameter on the constructor.
