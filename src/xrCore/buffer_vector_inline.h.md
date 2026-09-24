# src/xrCore/buffer_vector_inline.h

> How the caller-owned dynamic array moves its elements: what is constructed, what is destroyed, and in which order, for every size-changing operation.

**Needs** — [`buffer_vector.h`](buffer_vector.h.md)
**Used by** — [`buffer_vector.h`](buffer_vector.h.md)
**Tier floor** — T1: every operation is explicit in-place construction and destruction over raw memory.

## Purpose

Carries the bodies of the collection declared in [`buffer_vector.h`](buffer_vector.h.md). The split is a compilation concern; the substance is the element-lifetime discipline, which is the part a rebuild on a tier without explicit construction has to translate rather than copy.

## The discipline

Three rules govern everything here, and every operation is a consequence of them:

1. **Memory beyond the live range is raw.** Nothing between the end and the ceiling is a constructed object. Any operation that extends the live range must construct; any that shrinks it must destroy.
2. **Assignment destroys first, then constructs.** Assigning a new content set destroys the whole existing range before placing anything, rather than assigning element by element — so the element type needs no assignment, only construction and destruction.
3. **The bound is asserted, never enforced.** Every operation that moves the end checks it against the ceiling and asserts. In a shipping build the check vanishes.

## Operations, in terms of the discipline

| Operation | Elements constructed | Elements destroyed | Elements moved |
|---|---|---|---|
| construct empty | none | none | none |
| construct filled | the requested count | none | none |
| construct from a range | one per source element | none | none |
| assign | one per new element | the whole existing range, first | none |
| clear | none | the whole range | none |
| resize larger | the new tail, default-constructed | none | none |
| resize smaller | none | the removed tail | none |
| reserve | nothing at all — capacity is fixed | | |
| push at the end | one | none | none |
| pop at the end | none | one | none |
| insert at a position | the new elements | none | everything at and after the position shifts up |
| erase | none | the erased elements | everything after shifts down |
| destroy the collection | none | the whole range | none |

**Invariants** — Insertion shifts by assigning over existing objects and constructing only into the freshly exposed tail; erasure shifts down by assignment and destroys only the now-unused tail. That is the standard dynamic-array dance and its only unusual property here is that the tail's raw-versus-constructed boundary is the live end, not the capacity.

**Notes** — The type's own swap exchanges the three pointers, so swapping two of these collections swaps which *buffer* each refers to — which is only meaningful when both buffers outlive both collections. It is used where a caller holds two buffers and wants to exchange roles, never to move a collection.
