# src/xrPhysics/ode_include.h

> The single point at which the dynamics library is pulled in, so its scalar math can be replaced on the way.

**Needs** — [`ode_redefine.h`](ode_redefine.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`dCylinder.cpp`](dcylinder/dCylinder.cpp.md) · [`dTriBox.h`](tri-colliderknoopc/dTriBox.h.md)
**Tier floor** — T1: reaches into a foreign library's headers and overrides names it declared.

## Purpose

Every file in the module that needs the dynamics library goes through this one file rather than
naming the library directly. The reason is [`ode_redefine.h`](ode_redefine.h.md): the library's
scalar math must be swapped for the engine's *after* the library has declared it and *before* any
call site is compiled, and that ordering only holds if there is exactly one place where the library
enters.

That is the whole decision, and it survives a rebuild as a rule rather than a file: the dynamics
seam is reached through one adapter, never directly, so that anything which must be substituted on
the way in has a place to sit.
