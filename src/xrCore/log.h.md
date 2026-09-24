# src/xrCore/log.h

> Declares the logging surface implemented in [`log.cpp`](log.cpp.md).

**Needs** — [`log.cpp`](log.cpp.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md)
**Used by** — [`xr_dsa.cpp`](Crypto/xr_dsa.cpp.md) · [`StackTrace.cpp`](Debug/StackTrace.cpp.md) · [`FTimer.h`](FTimer.h.md) · [`LocatorAPI.cpp`](LocatorAPI.cpp.md) · [`ModuleLookup.cpp`](ModuleLookup.cpp.md) · [`_cylinder.cpp`](_cylinder.cpp.md) · [`_math.cpp`](_math.cpp.md) · [`dump_string.cpp`](dump_string.cpp.md) · [`log.cpp`](log.cpp.md) · [`os_clipboard.cpp`](os_clipboard.cpp.md) · [`xrCore.cpp`](xrCore.cpp.md) · [`xrCore.h`](xrCore.h.md) · [`xrDebug.cpp`](xrDebug.cpp.md) · [`xrMemory.cpp`](xrMemory.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration surface over line-oriented output.

## Purpose

Declares logging, described in [`log.cpp`](log.cpp.md). It is included by almost every file in the engine, which is why it forward-declares the vector and matrix types it logs rather than including their definitions.

## Exported units

- **`Msg`** — format and log. The entry point nearly every caller uses.
- **`Log`** — log one string; plus one overload per loggable value type (string, each integer width, real, 3-vector, 4x4 matrix), each rendering "message value".
- **`LogWinErr`** — log a message with a platform result code rendered as text.
- **`LogCallback`** — a function pointer plus an opaque context, with an emptiness test. Installed with `SetLogCB`, which returns the previous one so callbacks chain.
- **`CreateLog` / `CloseLog` / `FlushLog`** — open the file and drain the backlog; release; force to disk.
- **The line list** — exported directly, so the console can render scrollback without a copy. This is a deliberate hole in the encapsulation and the only reason the whole history is kept.
- **The callback-armed flag** — exported directly, so the console can mute itself while it is drawing without losing the callback.
- **Vector-splat helper** — expands a 3-vector into three arguments for formatted output.
