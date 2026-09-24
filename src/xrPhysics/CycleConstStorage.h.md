# src/xrPhysics/CycleConstStorage.h

> A fixed-length ring of samples, indexed from the newest backwards.

**Needs** — _(none)_
**Used by** — [`pose_extrapolation.h`](../xrGame/pose_extrapolation.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md)
**Tier floor** — T3: an array and a modulus.

## Purpose

Several physics decisions look at a short recent history — the last N positions, the last N
velocities — to decide whether motion is real or is solver noise. This is the container for
that: fixed capacity, no allocation, oldest sample silently overwritten.

## State

```text
RECORD CycleConstStorage<T, size>
  array : T[size]
  first : int        # index of the OLDEST slot, i.e. where the next write goes
  # index i names element (first + i) mod size, so index 0 is the oldest
```

## `push_back`

**Contract** — overwrites the oldest slot and advances. Constant time, never allocates,
never reports that anything was discarded.

## `operator[]`

**Contract** — reads the i-th oldest sample. Out-of-range indices wrap rather than fail, so
the caller is responsible for staying inside the capacity.

## `fill_in`

**Contract** — sets every slot to one value, used to prime the history at creation so that
the first few reads are not garbage. A rebuild that can express "empty but growing" instead
may prefer that; priming is simpler and gives the consumers a uniform window width from the
first step, which is what they want.

**Notes** — the history never records *when* a sample was taken. That is sound only because
every writer pushes exactly once per fixed physics step, so index distance is time. A
rebuild that pushes from the frame loop instead will get a history whose width varies with
frame rate.
