# src/xrCore/Debug/StackTrace.h

> Declares the two ways to ask for a call stack: from here, or from a thread frozen at a fault.

**Needs** — [`StackTrace.cpp`](StackTrace.cpp.md) · [`../../xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md) · [`../../xrCommon/xr_string.h`](../../xrCommon/xr_string.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`StackTrace.cpp`](StackTrace.cpp.md) · [`xrDebug.cpp`](../xrDebug.cpp.md)
**Tier floor** — T1: the second form takes a machine register snapshot, which is a platform concept with no portable shape.

## Purpose

Declares the surface implemented in [`StackTrace.cpp`](StackTrace.cpp.md). The two-form split is the load-bearing part: the crash handler cannot ask the faulting thread to walk its own stack, because by then it is running the handler's frames instead. So it captures the register state at the fault and hands *that* in.

## Exported units

- **`BuildStackTrace(max_frames = 512)`** — the current thread's stack, from the caller outward, as one formatted line per frame.
- **`BuildStackTrace(register_snapshot, max_frames)`** — the same for a thread whose registers were captured elsewhere. Only exists where the platform has such a snapshot.

**Notes** — the default of 512 frames is a cap on a diagnostic, not a correctness bound; the crash handler asks for 1024 because a stack overflow produces a trace of exactly one deeply repeated frame and the repetition is the evidence.
