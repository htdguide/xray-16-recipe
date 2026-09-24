# src/xrCommon/xr_stack.h

> A last-in-first-out view over the engine's growable array.

**Needs** — [`xr_vector.h`](xr_vector.h.md)
**Used by** — [`object_comparer.h`](../Common/object_comparer.h.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md)
**Tier floor** — T3: a naming convenience with one substantive default.

## Purpose

Exists to change one default. The standard stack adapter is backed by a double-ended queue;
this file re-backs it with the engine's growable array. That is the entire content of the
file and it is a real decision: the stacks in this engine are the recursive-descent work
stacks of the visibility walk and the collision-tree query, pushed and popped thousands of
times per frame, and a contiguous backing makes those pushes a bounds check and a store.

A rebuild deletes the file and uses its own array as a stack.

## `xr_stack`

**Contract** — push, pop, inspect the top. Backed by the contiguous growable array unless a
caller names another container.

**Notes** — the declaration shorthand declares a named stack type; it carries no meaning.
