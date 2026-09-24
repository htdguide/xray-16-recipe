# src/xrCore/buffer_vector.h

> A dynamic array over memory somebody else owns — the shape that lets per-frame and per-query collections live on the stack instead of the heap.

**Needs** — [`buffer_vector_inline.h`](buffer_vector_inline.h.md) · [`xrDebug_macros.h`](xrDebug_macros.h.md)
**Used by** — [`_stl_extensions.h`](_stl_extensions.h.md) · [`buffer_vector_inline.h`](buffer_vector_inline.h.md) · [`UIGameCTA.h`](../xrGame/UIGameCTA.h.md) · [`doors_actor.h`](../xrGame/doors_actor.h.md) · [`file_transfer.cpp`](../xrGame/file_transfer.cpp.md) · [`filereceiver_node.cpp`](../xrGame/filereceiver_node.cpp.md) · [`filereceiver_node.h`](../xrGame/filereceiver_node.h.md) · [`filetransfer_node.cpp`](../xrGame/filetransfer_node.cpp.md) · [`filetransfer_node.h`](../xrGame/filetransfer_node.h.md) · [`imotion_position.cpp`](../xrGame/imotion_position.cpp.md)
**Tier floor** — T1: it constructs and destroys elements in place in memory it did not allocate, and its capacity is fixed by the caller's buffer.

## Purpose

The frame budget forbids allocating thousands of short-lived collections — collision query results, visible-object lists, path candidate sets. This type gives those the full sequence interface over a buffer the caller supplies, usually from the stack. The capacity is fixed at construction and exceeding it is a failure rather than a reallocation; that is the trade the type exists to make.

## State

```text
RECORD BufferVector of T
  begin   : address       # first element; the caller's buffer, not owned
  end     : address       # one past the last live element
  ceiling : address       # begin + the capacity the caller declared
```

**Invariants**

- `begin <= end <= ceiling`. Asserted after every size-changing operation.
- **The memory is not owned.** Destruction destroys the live elements and nothing else; the buffer outlives the collection by construction.
- **Elements between `end` and `ceiling` are raw memory, not objects.** Growing constructs them; shrinking destroys them. This is what makes the type usable with element types that have real construction, and what makes it different from simply indexing an array.
- Capacity never changes. A request to reserve more is a no-op — not an error, a no-op — because the caller already chose the capacity.

## Exported units

The full sequence surface: construct empty, construct filled with a repeated value, construct from another collection or an iterator pair; assign from either; clear; resize; insert one, many or a range at a position; erase one or a range; push and pop at the end; indexed access with and without a bound check; front, back; forward and reverse iteration in both const and mutable forms; emptiness, size, capacity.

## Notes

The contracts are those of any dynamic array, with three differences a rebuild must carry:

- **Overflow is a failure, not a growth.** Every operation that would push `end` past `ceiling` asserts. In a shipping build the assertion is compiled out and the write happens anyway — so the capacity a caller declares must be a *proven* bound, not a guess. This is the single most important thing about the type.
- **Reserve does nothing.** Code ported from a growing array will silently lose its pre-sizing.
- **Copy-assignment copies contents into the existing buffer**, it does not adopt the source's buffer. Two collections over two buffers stay over two buffers.

The elementwise construction and destruction helpers are folded into the operations that use them; they are not part of the surface. See [`buffer_vector_inline.h`](buffer_vector_inline.h.md) for what each operation does to the element range.
