# src/xrScriptEngine/xrScriptEngine.hpp

> The module's own boundary header: the visibility marker every exported name carries, and the
> two guest interpreter types named without including the guest's headers.

**Needs** — [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)

**Used by** — [`BindingsDumper.hpp`](BindingsDumper.hpp.md) · [`ScriptEngineScript.hpp`](ScriptEngineScript.hpp.md) · [`ScriptExporter.hpp`](ScriptExporter.hpp.md) · [`pch.hpp`](pch.hpp.md) · [`script_callStack.hpp`](script_callStack.hpp.md) · [`script_debugger_messages.hpp`](script_debugger_messages.hpp.md) · [`script_debugger_threads.hpp`](script_debugger_threads.hpp.md) · [`script_engine.hpp`](script_engine.hpp.md) · [`script_lua_helper.hpp`](script_lua_helper.hpp.md) · [`script_process.hpp`](script_process.hpp.md) · [`script_profiler.hpp`](script_profiler.hpp.md) · [`script_space.hpp`](script_space.hpp.md) · [`script_stack_tracker.hpp`](script_stack_tracker.hpp.md) · [`script_thread.hpp`](script_thread.hpp.md) · _and 1 more_

**Tier floor** — T4: it declares nothing but a linkage decision and two opaque names.

## Purpose

Two jobs, both incidental to what the module *does* but load-bearing for how it is assembled.

First, the module can be built into the executable or as a separately loaded library, and names
crossing that boundary must be marked. A rebuild whose modules are linked one way only deletes
this entirely.

Second — and the part worth keeping — it declares the interpreter's state and frame-description
types as **opaque names**. Every header in this module that mentions the VM mentions it through
these, so the guest interpreter's own headers are pulled in only by the handful of files that
actually manipulate its stack. That is a deliberate containment: it is what lets the rest of the
engine hold a script engine without acquiring a dependency on the interpreter, and it is the
reason a rebuild can swap the interpreter with a bounded blast radius.

## Exported units

- The module visibility marker.
- `lua_State` — an opaque handle to an interpreter or coroutine.
- `lua_Debug` — an opaque handle to a frame description.
