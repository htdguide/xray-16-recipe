# src/xrCommon/xr_smart_pointers.h

> Owning handles whose release runs the object's destruction and returns the memory to the engine allocator, not to the platform's.

**Needs** — [`xrCore/xrMemory.h`](../xrCore/xrMemory.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the whole file is about *when* and *where* an object's memory is released, which is the constraint that keeps the engine off a collected tier.

## Purpose

The engine allocates objects through its own policy, so it must also release them through
that policy. Ordinary owning handles release through the platform's, which would hand a
block from one heap to another. This file defines the release behaviour and then names two
handle kinds that use it, plus their two construction helpers.

A rebuild in a language with automatic lifetime deletes the file. What it must preserve is
the *ordering* requirement, not the mechanism: graphics, audio and script handles have to
be released at a defined point in a defined order, so the objects that own them must have
a destruction step that runs deterministically — not whenever a collector gets to it. A
collected tier expresses this with explicit scopes, and this file becomes those scopes.

## The release step

**Contract** — releasing an object runs its destruction logic and then returns its storage
to the engine allocator.

**Invariants**

```text
# The address you hold is not always the address the allocator gave out.
#   When a type is reached through one of several bases it inherits, the handle
#   points at a sub-object partway into the block. Releasing must first recover
#   the start of the most-derived object and free THAT address; freeing the
#   sub-object address corrupts the heap. The engine does this recovery for every
#   type that has a runtime type identity, and skips it (as an optimization) for
#   types that do not.
#   A rebuild in a language without multiple inheritance never meets this. A
#   rebuild that has it, and whose allocator cannot be handed an interior
#   pointer, must reproduce the recovery.

# Release is idempotent against a null handle and clears the handle it was given.
#   The engine relies on "release then check" being safe during teardown, where
#   the same object is reachable from several registries that unregister in an
#   order nobody fixed.
```

## `xr_unique_ptr` — single ownership

**Contract** — exactly one handle owns the object; releasing the handle releases the object
through the step above. Ownership transfers on move; it cannot be copied. Costs nothing
beyond the pointer itself.

## `xr_shared_ptr` — shared ownership

**Contract** — several handles own the object jointly; the object is released when the last
one goes away. Costs a separately-allocated reference count alongside the object.

**Notes** — the count block is allocated through the global construction operator, which is
itself replaced process-wide, so shared ownership does not leak out of the engine's heap
even though this file never says so. A rebuild that keeps a separate count must route it
the same way, or the memory report under-counts.

## `xr_make_unique` / `xr_make_shared`

**Contract** — construct an object of a named type from a forwarded argument list, through
the engine allocator, and return it already owned by the corresponding handle. These exist
so that no call site writes a raw allocation followed by a handle construction, which is
the shape that leaks when construction fails between the two.
