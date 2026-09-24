# src/xrCore/Memory — routing every allocation through one place

Part of chapter 6, [`src/xrCore`](../README.md). Three files. The allocator itself is
[`../xrMemory.cpp`](../xrMemory.cpp.md), one level up; this directory holds the two things
that sit on top of it.

## What this module is responsible for

**Making the language's containers allocate through the engine's allocator.** Every container
alias in [`src/xrCommon`](../../xrCommon/README.md) is the standard container parameterized
on the adapter in [`xalloc.h`](xalloc.h.md). That single mechanism is the only reason
container memory is visible to the engine's accounting at all.

**Aligned allocation on top of an allocator that does not align.** Four-wide float values
must sit on 16-byte boundaries to be loaded and stored as one register, and a few device-
facing buffers want more. The underlying allocator promises nothing stronger than the
platform default, so this directory builds the guarantee by over-allocating and recording
the real address just below the aligned one.

## Where it sits

It rests on [`../xrMemory.h`](../xrMemory.h.md) and, through it, on
[Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator). Everything from chapter 2
onward consumes it, almost always without knowing: a container declaration is an allocation
decision.

## The load-bearing ideas

**The allocator is stateless and any two instances are interchangeable.** That is the
decision that makes containers movable, swappable and spliceable with no
allocator-propagation question ever arising. A rebuild that gives its allocator state inherits
that whole problem for no gain.

**Allocation failure is not an error path.** The underlying allocator either succeeds or
terminates the process, so nothing in the engine checks an allocation result. A rebuild may
reasonably disagree, but it must then add the checks at every one of several thousand call
sites, and the twins say so rather than pretending the checks exist.

**The over-allocation trick is a T1 artifact with a T2 answer.** Stashing the original
address in the bytes immediately below the aligned one works, and it is what a language with
no aligned-allocation primitive must do. A language that has one deletes
[`xrMemory_align.cpp`](xrMemory_align.cpp.md) entirely — but must keep the *requirement*,
because the alignment is what the four-wide math layer and the graphics buffers depend on.

## The twins

| File | Role |
|---|---|
| [`xalloc.h`](xalloc.h.md) | **The adapter** that routes every container's storage through the engine's allocator. Stateless, always-equal. Substantive. |
| [`xrMemory_align.h`](xrMemory_align.h.md) | Declares aligned allocation, reallocation, release and size query. |
| [`xrMemory_align.cpp`](xrMemory_align.cpp.md) | **Aligned allocation built on an unaligned allocator**: over-allocate, stash the real address below the aligned one, recover it on release. Substantive. |
