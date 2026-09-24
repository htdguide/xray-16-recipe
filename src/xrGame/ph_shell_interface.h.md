# src/xrGame/ph_shell_interface.h

> The one-method interface an object implements to say "I know how to build my own physics body".

**Needs** — [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHShellCreator.h`](PHShellCreator.h.md) · [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md)
**Tier floor** — T3: an interface declaration

## Purpose

An interface with a single demand, in its own file so that anything able to construct a
physics body can be named without dragging in the object hierarchy it lives in.

## State

`Stateless.`

## `IPhysicShellCreator`

**Contract** — an implementor must provide one operation: *create your physics shell*. It
takes nothing and returns nothing; the implementor knows its own model, its own mass
properties and its own joint configuration, and the result is attached to the implementor
rather than handed back.

**Invariants** — the operation is a *command*, not a factory, and that is the load-bearing
choice. A physics shell must be registered with the physics world, bound to the object's
bones and reachable from the object in the same step, so returning a detached shell for a
caller to install would create a window in which the shell exists and is not attached. Any
rebuild must keep construction and attachment in one operation.

The interface says nothing about idempotence. Callers are expected to know whether a shell
already exists; the implementors decide what a second call means.

**Notes** — an interface this small exists to break a dependency, not to abstract a family.
The callers are lifecycle code — spawn, level load, a script order — that must trigger body
creation without knowing what kind of object it is holding.
