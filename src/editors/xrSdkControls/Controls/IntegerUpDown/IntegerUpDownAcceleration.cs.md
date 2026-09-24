# src/editors/xrSdkControls/Controls/IntegerUpDown/IntegerUpDownAcceleration.cs

> One row of a spin box's acceleration table: after this long, step by this much.

**Needs** — [`IntegerUpDown.cs`](IntegerUpDown.cs.md)
**Used by** — [`IntegerUpDown.cs`](IntegerUpDown.cs.md) · [`IntegerUpDownAccelerationCollection.cs`](IntegerUpDownAccelerationCollection.cs.md)
**Tier floor** — T3: two checked numbers.

## Purpose

The unit of the acceleration schedule described in [`IntegerUpDown`](IntegerUpDown.cs.md). It is a pair with a validity rule, and it is a separate type only because the collection that sorts these needs something to sort.

## State

```text
RECORD Acceleration
  seconds   : int    # invariant: >= 0 — how long the button must be held for this step
  increment : int    # invariant: >= 0 — the step size once it is
```

**Invariants** — both fields are non-negative, checked on construction and on every write. A negative hold time has no meaning; a negative increment would silently reverse the direction of the spin button, which is worse than rejecting it.

## Notes

Both are whole seconds, not a finer unit. That makes the coarsest useful schedule — a second of hold before the first promotion — the shortest expressible one, and a rebuild wanting a snappier ramp needs a finer unit here.
