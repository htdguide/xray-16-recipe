# src/xrGame/ik/aint.h

> Declares an interval on a circle and a set of them — the representation the joint-limit
> machinery answers in.

**Needs** — _(none)_
**Used by** — [`aint.cxx`](aint.cxx.md) · [`eqn.h`](eqn.h.md) · [`jtlimits.cxx`](jtlimits.cxx.md) · [`limb.cxx`](limb.cxx.md) · [`limb.h`](limb.h.md)
**Tier floor** — T2. The set is a linked list whose nodes are allocated during a solve.

## Purpose

Declares the surface implemented in [`aint.cxx`](aint.cxx.md), which carries the
substance. Several predicates are inline here because they are one comparison each; their
tolerances, however, are load-bearing and are described in the implementation twin.

## Exported units

The tolerance pair every operation in the directory is graded against: a tight one used
for containment and equality, and a loose one — a thousand times larger — used when
deciding whether two intervals should be treated as one.

Comparison helpers that admit a tolerance: equality, is-zero, is-a-full-turn,
less-or-equal, greater-or-equal. And the shortest angular distance between two angles,
measured whichever way around is nearer.

**An interval on a circle**, given by a low and a high angle both normalized into a single
turn. `low > high` is legal and means the interval wraps through zero — that is the whole
difficulty of the type. Its operations: set either end, read either end, is it the full
circle, is it empty, does it contain an angle, its angular size, its midpoint, subset and
superset tests, the signed distance from an angle to it, and a split of a wrapping
interval into two non-wrapping ones.

**An iterator over one interval** that yields a requested number of angles spread across
it, inset from both ends, or — reversed — across its complement.

**A set of intervals**, held as a list. Its operations: clear, copy, add an interval with
merging, add every interval of another set, is it empty, which member is largest, does any
member contain an angle, distance from an angle to the nearest member, count of members,
a map over the members, an iterator, and a repair pass that rejoins a member ending at the
end of the turn with one starting at the beginning.

Set union and set intersection as free functions.
