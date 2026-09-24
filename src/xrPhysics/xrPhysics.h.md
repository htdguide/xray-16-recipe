# src/xrPhysics/xrPhysics.h

> Declares which symbols of the physics module are visible to the rest of the engine.

**Needs** — _(nothing)_
**Used by** — [`PHUpdateObject.h`](PHUpdateObject.h.md) · [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`StdAfx.h`](StdAfx.h.md) · [`console_vars.h`](console_vars.h.md) · [`debug_output.h`](debug_output.h.md) · [`icollisiondamagereceiver.h`](icollisiondamagereceiver.h.md) · [`phvalide.h`](phvalide.h.md) · [`xrPhysics.cpp`](xrPhysics.cpp.md)
**Tier floor** — T1: exists only because the physics module can be built as a separately loaded binary whose exported symbol set must be declared at compile time.

## Purpose

The physics module is one of the units that may be loaded dynamically at start-up rather than linked
into the executable (see [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)).
This file names the one decision that follows: a symbol is either part of the module's public
surface or it is private to the module.

The whole file is incidental. In a rebuild, whatever the language calls "public to other modules"
replaces it, and the file disappears. What must survive is the *list* of things marked public —
the factories and interfaces in [`PhysicsShell.h`](PhysicsShell.h.md), the validity helpers in
[`phvalide.h`](phvalide.h.md), the tuning block in [`console_vars.h`](console_vars.h.md) and the
collision-damage entry points in [`icollisiondamagereceiver.h`](icollisiondamagereceiver.h.md).
Everything else in this directory is module-private and a rebuild is free to restructure it.

## `XRPHYSICS_API`

**Contract** — marks a declaration as part of the module's exported surface. Resolves to nothing at
all when the whole engine is built as one binary; to "export" when compiling this module; to
"import" when compiling a module that uses it.
