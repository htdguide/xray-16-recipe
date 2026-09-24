# src/xrCore/xrDebug.h

> Declares the failure machinery implemented in [`xrDebug.cpp`](xrDebug.cpp.md), and the two interfaces the failure path needs another module to supply.

**Needs** — [`xrDebug.cpp`](xrDebug.cpp.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md) · [`xr_types.h`](xr_types.h.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`entry_point.cpp`](../utils/mp_balancer/entry_point.cpp.md) · [`main.cpp`](../utils/xrCompress/main.cpp.md) · [`matrix.cpp`](../utils/xrMiscMath/matrix.cpp.md) · [`vector.cpp`](../utils/xrMiscMath/vector.cpp.md) · [`PPInfo.cpp`](PostProcess/PPInfo.cpp.md) · [`StringConversion.cpp`](Text/StringConversion.cpp.md) · [`ScopeLock.cpp`](Threading/ScopeLock.cpp.md) · [`Task.hpp`](Threading/Task.hpp.md) · [`TaskManager.cpp`](Threading/TaskManager.cpp.md) · [`_math.cpp`](_math.cpp.md) · [`_quaternion.h`](_quaternion.h.md) · [`_random.h`](_random.h.md) · [`clsid.cpp`](clsid.cpp.md) · [`dump_string.cpp`](dump_string.cpp.md) · _and 8 more_
**Tier floor** — T1: it declares callbacks that run inside a signal handler and hands out a raw window handle to the platform layer.

## Purpose

Declares the failure surface described in [`xrDebug.cpp`](xrDebug.cpp.md). Its own substance is the pair of abstract interfaces the core demands of whoever owns the window and the user's settings — these are real contracts a rebuild must satisfy, not declarations.

## `WindowHandler` — required of whoever owns the window

**Contract** — The core cannot depend on the engine or the renderer, but it must not pop a modal box over a fullscreen surface, and it must be able to get rid of that surface before breaking into a debugger. So it demands four operations from whoever owns the window:

- **give me the native window handle** — so a platform message box can be parented to it;
- **give me the window object** — so the portable message box can be parented to it;
- **a dialog is opening / a dialog has closed** — called on both edges, inside the failure lock, so the implementor can drop out of fullscreen and restore afterwards;
- **this is fatal** — called when execution will not resume; the implementor must tear the window down, because what follows is a debugger break or a process kill and a live fullscreen surface would leave the display unusable.

**Invariants** — Every one of these may be called from a signal handler, from a worker thread, and with the heap in an unknown state. None may allocate, block on anything the main thread holds, or fail.

## `UserConfigHandler` — required of whoever owns the settings file

**Contract** — One operation: name the file the user's settings live in. The crash reporter attaches it, and the core does not otherwise know that such a file exists. Exists because the settings filename differs between the game and the tools.

## Exported units

- **`ErrorLocation`** — file, line, function, captured at the assertion site.
- **`AssertionResult`** — the three outcomes plus "undefined" (the dialog itself failed) and "ok" (a message with no choice).
- **Initialize / finalize** — install and remove the handlers for the process.
- **On thread spawn / on thread exit** — install and remove them for one thread.
- **On filesystem initialized** — register the log and the report directory, once paths resolve.
- **Debugger-present probe** — changes whether a failure breaks or reports.
- **Processing-failure probe** — true while the failure lock is held, so code reachable from the failure path can avoid recursing.
- **Handler accessors** — set and get the two interfaces above and the out-of-memory hook.
- **Gather info** — format a failure into a buffer, log it, capture a trace, put it on the clipboard.
- **Fatal** — format a message and fail unconditionally.
- **Fail** — the assertion entry point, in three flavours: description text, a platform result code to be rendered as text, or a built string.
- **Do exit** — a clean refusal to continue; never returns.
- **Show message** — one box, either informational or three-way.
- **Log stack trace** — capture and log without failing.
- **Error-code-to-text** — render a platform result code; yields nothing on platforms without one.
- **Set bug-report file** — name an extra attachment for the crash report.
- **`make_string`** — format into a scratch buffer and return an owning string, for building assertion descriptions at the call site.
