# src/xrPhysics/PHShellNetState.cpp

> A multi-body object's network state is nothing but its bodies' states, in element order.

**Needs** — [`PHShell.h`](PHShell.h.md) · [`PHElement.h`](PHElement.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md)
**Used by** — [`PHShell.h`](PHShell.h.md)
**Tier floor** — T3: an ordered walk over a list.

## Purpose

The decision recorded here is that **a shell has no network state of its own**. Its joints are
derived from its elements' placements, its island membership is local, and its object-in-root
transform is authored data both ends already hold. Everything that must cross the wire is in the
bodies.

## `net_Export` / `net_Import`

**Contract** — walk the element list in order, exporting or importing each element's state (see
[`PHElementNetState.cpp`](PHElementNetState.cpp.md)). No length prefix and no element identifiers
are written.

**Invariants** — **both ends must have built the shell from the same model**, so that element count
and element order match exactly. This is the load-bearing assumption of the whole scheme: the
stream is positional, so a receiver whose shell has one element more or fewer does not detect the
mismatch — it silently assigns every body the wrong state and the object flies apart. A rebuild that
wants to survive model mismatches must add a count and per-element identity; the original relies on
both ends running the same game data, which the
[conformance criteria](../../SYSTEM-REQUIREMENTS.md#6-conformance) assume anyway.

**Notes** — the cost of this is that a ragdoll's full state is sixteen or so bodies' worth of
position, orientation, two velocities, force, torque and a previous pose every time it is
synchronized. That is why ragdolls are synchronized rarely and settle locally, rather than being
streamed; the sleep flag in each element's state is what tells a receiver it may stop expecting
updates.
