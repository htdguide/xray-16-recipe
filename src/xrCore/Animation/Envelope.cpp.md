# src/xrCore/Animation/Envelope.cpp

> Curve editing: finding, inserting, deleting, retiming and simplifying keys.

**Needs** — [`Envelope.hpp`](Envelope.hpp.md) · [`interp.cpp`](interp.cpp.md) · [`../FS.h`](../FS.h.md)
**Used by** — [`Envelope.hpp`](Envelope.hpp.md)
**Tier floor** — T2: list manipulation over a sorted sequence of keys.

## Purpose

The operations on an animation curve. The curve's data and its on-disk form are in [`Envelope.hpp`](Envelope.hpp.md); its evaluation is in [`interp.cpp`](interp.cpp.md). Most of what is here is authoring, but two things reach the engine: the sorted-order invariant that every insertion preserves, and the simplification pass that shrinks a constant channel to two keys, which is applied to shipped animation and therefore shapes what the loader sees.

## `FindKey` / `FindNearestKey`

**Contract** — `FindKey` returns the key at a given time within a tolerance, or nothing. `FindNearestKey` returns the two keys bracketing a time: the last key at or before it and the first key after it. Both walk the sorted list forwards and stop at the first key later than the target, so both are linear and both rely on the ordering.

**Invariants** — when the time lands exactly on a key, `FindNearestKey` returns *that* key as the upper bracket's predecessor, which makes the bracket asymmetric at exact hits. Callers that scale a range depend on that: the range's own endpoints must be inside the affected span.

Both clamp at the ends rather than failing: a time before every key yields the first key as both brackets, and a time after every key yields the last.

## `InsertKey`

**Contract** — set the value at a time. If a key already exists there within a tolerance, overwrite its value and return; otherwise create a key with the default spline shape and splice it in at the position that preserves the ordering.

**Invariants** — creating a key also forces **both end behaviours to hold-the-nearest-value**. That is a side effect on the whole curve from a per-key operation, and it is deliberate: a curve being authored key by key should not extrapolate wildly off either end while it is half-built.

## `DeleteKey`

**Contract** — remove the key at a time within the tolerance, if there is one. No renumbering is needed; the list stays sorted by construction.

## `ScaleKeys`

**Contract** — stretch or compress a time range by a factor, and slide everything after it so the curve stays continuous. Returns whether anything was done.

```text
FUNCTION scale_keys(e, from_time, to_time, factor, tolerance) -> bool
  lo := key at from_time, or the bracket below it
  hi := key at to_time,   or the bracket above it
  IF lo IS none OR lo = hi THEN RETURN false
  IF hi EXISTS THEN hi := the key after hi          # make the range inclusive

  # Inside the range: each interval is scaled; `shift` accumulates how much
  # later everything downstream must move.
  shift := 0
  previous_time := time_of(lo)
  FOR EACH k FROM lo+1 UNTIL hi
    new_time := shift + previous_time + (k.time - previous_time) * factor
    shift    := shift + ((new_time - time_of(k-1)) - (k.time - previous_time))
    previous_time := k.time                          # the ORIGINAL time
    k.time   := new_time

  # After the range: everything slides rigidly by the accumulated shift.
  FOR EACH k FROM hi UNTIL end
    new_time := shift + k.time
    shift    := shift + ((new_time - time_of(k-1)) - (k.time - previous_time))
    k.time   := new_time
  RETURN true
```

**Invariants** — the previous time must be captured *before* the key is overwritten, since the following key's interval is measured against the original spacing. That is the only subtle thing in the function and is the usual way this operation is got wrong.

**Notes** — the accumulator is recomputed on every key rather than being the simple constant it mathematically is. The extra arithmetic is harmless. The second loop carries the first loop's final previous-time value into its own accumulator update, which makes its shift drift; in practice the second loop's shift is only ever added to already-shifted values and the drift cancels. This is fragile and a rebuild should simply add one constant offset to every key after the range.

## `GetLength`

**Contract** — the span from the first key's time to the last, optionally reporting both endpoints. An empty curve has length zero and both endpoints zero.

## `RotateKeys`

**Contract** — add a constant to every key's value. Named for its use: shifting a rotation channel by an angle. Does not touch times or shapes.

## `Optimize`

**Contract** — if every key in the curve is *equivalent* to the first, and there are more than two, replace the whole list with copies of the first and last keys. Equivalence compares value, shape, tension, continuity, bias and all four handle parameters within a floating-point tolerance — **but not time**, which is what makes this a constant-channel test rather than a duplicate-key test.

```text
FUNCTION optimize(e)
  IF e.keys IS EMPTY THEN RETURN
  reference := e.keys[0]
  IF ANY k IN e.keys IS NOT equivalent_to(reference) THEN RETURN
  IF count(e.keys) <= 2 THEN RETURN
  e.keys := [copy(first), copy(last)]
```

**Invariants** — the last key is kept rather than dropping down to one key, because a curve's *length* is defined by its endpoints and the animation system reads that length. Collapsing to a single key would make every such channel zero-length.

**Notes** — the equivalence test compares the shape field with a floating-point near-equality, although the shape is an integer code. Distinct shapes differ by one, and the tolerance is far smaller, so it works — but it is comparing an enumeration as a real, and a rebuild should compare it as an integer.

This pass is why most bone channels in shipped animation banks have exactly two keys: a bone that does not move on one axis is stored as two identical keys, not as a flag. The loader has no "channel absent" case to handle because of it.

## `Evaluate`

Delegates to the evaluator contracted in [`interp.cpp`](interp.cpp.md).

## Lifetime

**Contract** — the curve owns its keys. Copy-constructing a curve deep-copies every key. Clearing destroys the keys but leaves the list's length; the separate clear-and-free form also empties the list.

**Notes** — the copy constructor first copies the whole object (which copies the list of references) and then replaces each entry with a fresh copy. The intermediate state where both curves reference the same keys is transient and never observed, but it is a trap: an exception in the middle leaves two curves owning one set of keys. In a rebuild with value semantics the problem does not exist.
