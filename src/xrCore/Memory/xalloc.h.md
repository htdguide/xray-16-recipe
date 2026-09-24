# src/xrCore/Memory/xalloc.h

> The adapter that makes every container in the engine allocate through the engine's allocator instead of the language's.

**Needs** — [`xrMemory.h`](../xrMemory.h.md) · [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`xr_allocator.h`](../../xrCommon/xr_allocator.h.md)
**Tier floor** — T1: it exists only because the language's containers default to a allocator the engine does not control. A tier where allocation is not the caller's business deletes this file outright.

## Purpose

Every container type alias in [`src/xrCommon`](../../xrCommon/README.md) — the vectors, maps, sets and strings the whole engine uses — is the standard container parameterized on this allocator. That is the single mechanism by which the engine's memory accounting sees container memory at all: without it, container storage would come from the language runtime's heap and be invisible to the budget, the leak tracker and the pooled small-block path.

## State

Stateless. The allocator carries no per-instance data at all, which is the decision that matters: any two instances are interchangeable, so containers may be moved, swapped and spliced freely without an allocator-propagation question ever arising. A rebuild that gives its allocator state inherits that whole problem.

## The allocator contract

**Contract** — allocate a block sized for a given count of a given element type and return it; release a block given its address, ignoring the count; construct an element in place; destroy one in place; report a nominal maximum count. Allocation failure is not reported here — the underlying allocator either succeeds or terminates the process, so there is no error path to propagate. Alignment is whatever the underlying allocator guarantees (sixteen bytes, per the seam), which is at least as strong as any element requires.

**Invariants** — two instances always compare equal, regardless of the element type they were parameterized on. That equality is what lets a container release memory that a differently-typed instance allocated, which happens routinely as containers rebind the allocator to their internal node types.

**Notes** — the maximum-count report clamps to at least one, so that an element type larger than the address space still yields a legal answer rather than zero. Nothing depends on the value; it exists because the container interface demands it.

The release path accepts either a typed or an untyped address, which is a concession to containers that free node memory through a generic pointer. In a rebuild there is one release and no such distinction.

The whole file is incidental in the sense the brief means: it is the shape of "route container memory through our allocator" in one particular language. What survives is the requirement — *all* bulk memory, including the memory containers allocate behind your back, must come from the engine's allocator so that the budget in [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator) is a real budget.
