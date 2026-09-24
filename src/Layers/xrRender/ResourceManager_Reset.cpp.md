# src/Layers/xrRender/ResourceManager_Reset.cpp

> The device-lost bracket: what every device-owned resource must release when the graphics device is torn down, and the order in which it is all rebuilt.

**Needs** — [`ResourceManager.h`](ResourceManager.h.md) · [`SH_RT.h`](SH_RT.h.md) · [`SH_Atomic.h`](SH_Atomic.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`tss.h`](tss.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it exists because device objects have a lifetime the language cannot see, and because the *order* of their recreation changes the result.

## Purpose

A graphics device can be lost — a resolution change, a mode switch, a driver reset — and every object it allocated becomes invalid. The seam requires a filling to support "a device-lost path that rebuilds every resource". This file is the registry's half of that path: a `begin` that releases exactly the device-owned objects, and an `end` that rebuilds them in an order that matters.

It is a separate file because it is the only place that knows *which* of the registry's resources are device-owned and which are plain data. That distinction is worth isolating.

## State

`Stateless`; it operates over the registry's collections. It relies on two remembered handles that live in the renderer rather than here: the previous dynamic vertex and index buffers, and the previous quad index buffer, kept alive across the bracket only so that stale references can be recognized (see below).

## `reset_begin`

**Contract** — release every device-allocated object the registry owns, leaving the descriptions behind. Must run before the device is destroyed. Does not touch textures, which the backend handles separately, nor the shader programs, which the backend rebuilds.

```text
FUNCTION reset_begin()
  FOR EACH state block
    release its device object, keep its recorded settings
  FOR EACH render target
    release it, keep its name, dimensions and format
  remember the current quad index buffer, then release it
  release the dynamic index stream
  release the dynamic vertex stream
```

**Invariants** — every state block keeps its *recorded settings* — the list of state assignments that produced it. That is the whole reason state blocks can be rebuilt at all: the block is a compiled artifact, and the recording is its source. A rebuild must keep the source of every compiled device object, not the object.

## `reset_end`

**Contract** — rebuild everything `reset_begin` released. Runs after the device is up again. The order is load-bearing in two places.

```text
FUNCTION reset_end()
  rebuild the dynamic vertex stream
  rebuild the dynamic index stream
  evict                                    # a no-op in the shipped build
  rebuild the shared quad index buffer

  # 1. repair geometry records that pointed at the dynamic buffers
  FOR EACH geometry
    IF its vertex buffer was the old dynamic vertex buffer
      point it at the new one
    IF its index buffer was the old dynamic index buffer
      point it at the new one
    ELSE IF its index buffer was the old quad index buffer
      point it at the new quad index buffer

  # 2. rebuild render targets in their original creation order
  FOR EACH render target IN ascending creation order
    recreate it

  # 3. re-record every state block from its recorded settings
  FOR EACH state block
    record its settings into a fresh device object
```

### Why geometry records need repairing

A geometry record is a triple of (vertex declaration, vertex buffer, index buffer). Most geometries name *static* buffers, which the level owns and which survive whatever the device did. But geometries that draw from the dynamic streams — every batched draw in the renderer — name the stream's buffer, and that buffer was just destroyed and recreated at a new address. Matching against the remembered old handles and substituting the new ones is how they are found.

The chain of two tests, with the second an `else`, is deliberate and the source says so: the quad index buffer and the dynamic index buffer are both candidates, and a geometry can only have come from one of them. Testing both independently would let a geometry be repaired twice if the allocator happened to hand back the same address.

**Notes** — The alternative a rebuild should consider is indirection: let a geometry name the *stream* rather than its current buffer, and the repair disappears. The original resolves the handle eagerly because doing it per draw would cost a pointer chase in the innermost loop of the frame.

### Why render targets must be recreated in creation order

Render targets are recreated sorted by a recorded creation ordinal, not in map order and not in name order. The registry's map is keyed by name, so iterating it would recreate them alphabetically — a different order from the first time.

The reason the order matters is video memory layout. These are large allocations made back to back at startup; recreating them in a different order can fragment differently, and on a constrained device the last one can fail to fit even though the same set fitted a moment ago. Preserving the original order makes the second allocation sequence identical to the first, which makes "it fitted before" a guarantee that it fits again.

A rebuild on an API with explicit memory placement has a better answer and should use it. A rebuild on any API still benefits from the property: *device resources are recreated in creation order*, so a reset is a replay of startup rather than a fresh attempt.

## `Dump`

**Contract** — writes counts for every map and list in the registry, and optionally each resource's name and current reference count. Console-triggered, and run automatically at destruction in non-shipping builds.

**Notes** — Running it from the destructor is a leak report: anything still holding a reference at that point is a bug, and the dump is the only evidence of it. The shipping build omits it because the process is about to exit anyway.

## Destructor

**Contract** — releases the pinned texture set, then dumps. The pinned set must go first, or every pinned texture appears in the leak report as a false positive.
