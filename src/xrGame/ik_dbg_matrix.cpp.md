# src/xrGame/ik_dbg_matrix.cpp

> Captures every intermediate transform of a foot-placement solve into a bounded history.

**Needs** — [`ik_dbg_matrix.h`](ik_dbg_matrix.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md) · [`ik_calculate_data.h`](ik_calculate_data.h.md)
**Used by** — reached through its declarations in [`ik_dbg_matrix.h`](ik_dbg_matrix.h.md); callers name that, not this file.
**Tier floor** — T3: instrumentation

## Purpose

Compiled out in every shipped configuration. A rebuild may omit the file; the twin exists to
keep the mirror complete and to record the one number that outlives the switch.

## State

```text
history_length : int = 130   # how many past solves are retained
```

**Notes** — the retention length is a bare constant with no derivation. A hundred and thirty
frames is roughly two seconds at the engine's frame rate, which is about the length of a
visible foot-sliding artifact, but nothing in the source says so.

## `next_state`

**Contract** — pushes the current snapshot into the history, evicting the oldest once the
history is full, then recaptures every transform from the limb's current solve.

The capture branches on which of the limb's two end bones the goal is expressed against, and
converts the goal into the other bone's frame using the limb's own bone-to-bone transform, so
that both are always recorded regardless of which one the solver chose. That conversion is
the only non-trivial thing here, and it is the same conversion the solver itself performs.

**Notes** — in both branches the final "start" relation is computed from the **goal**
transforms rather than the start ones — the same expression is used for both. That is a
copy-paste error in instrumentation, harmless and not worth reproducing.

## `next_goal`

**Contract** — records the goal as it was requested and as the limb resolved it, plus the
owner's transform at the moment of application. Called after the solve rather than during it,
which is what makes the before/after pair of owner transforms meaningful.
