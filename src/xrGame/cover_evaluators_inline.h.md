# src/xrGame/cover_evaluators_inline.h

> The evaluators' lifecycle operations and, more importantly, the rule by which a previous answer stays valid.

**Needs** — [`cover_evaluators.h`](cover_evaluators.h.md)
**Used by** — [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`cover_evaluators.h`](cover_evaluators.h.md)
**Tier floor** — T3: field assignment with one real decision

## Purpose

Supplies the constructors, accessors and setup paths for
[`cover_evaluators.h`](cover_evaluators.h.md). Most of it is initialization, but the setup
paths carry the **actuality rule**, which is the cover system's main cost control and is
written down nowhere else.

## State

`Stateless.` Operates on the evaluator fields described in
[`cover_evaluators.cpp`](cover_evaluators.cpp.md).

## the actuality rule

**Contract** — each `setup` compares every parameter against the one from last time and clears
actuality if it differs. Actuality survives only if *every* parameter is unchanged.

```text
FUNCTION setup(...parameters)
  mark initialized
  FOR EACH parameter
    actuality = actuality AND (new value is similar to the stored one)
    store the new value
```

**Invariants** — actuality is only ever *cleared* here; it is restored to true by `finalize`,
at the end of a pass. So the flag means "nothing changed between the previous pass and this
one", and an evaluator that is still actual may return its previous answer without offering a
single candidate. That is what keeps a creature standing still from re-searching for cover
every decision cycle.

**Notes** — the comparisons use approximate equality, and the ambush evaluator's enemy position
uses a **five-metre** tolerance: an enemy who has moved less than five metres does not
invalidate an ambush position. A commented-out ten-metre tolerance on the close-to-enemy
evaluator's enemy position shows the same idea was tried and abandoned there — that evaluator
now treats any enemy movement as invalidating, which is correct for an evaluator whose whole
score is the distance to the enemy.

**Notes** — the ambush evaluator deliberately does **not** compare its own position, though the
disabled line is still there. An ambusher that shifts slightly keeps its chosen spot.

## `initialize`

**Contract** — fixes the creature's current position for this pass, remembers the previous
answer, clears the running best to a large finite sentinel, drops any smart-cover firing
position, and stamps the update time — **unless** the call is marked fake, in which case the
timestamp is left alone.

**Invariants** — the fake call exists so a caller can prepare an evaluator (to ask it a
question, or to set it up for a later pass) without restarting the inertia hold. Stamping the
time on a preparatory call would make every such preparation extend the period during which
the creature refuses to look for better cover.

**Notes** — the close-to-enemy evaluator extends initialization by caching the creature's
*current* distance to the enemy, which its band tests compare against. Caching it once per
pass rather than recomputing per candidate is the difference between one distance computation
and several hundred.

## the remaining operations

**Contract** — `selected` and `loophole` read the answer; `set_inertia` sets the hold period;
`initialized`, `actual`, `invalidate` and `best_value` expose the pass state; `accessible`
defers to the creature's restrictors, answering yes when there are none; the four smart-cover
permission accessors read and write the two flags. `finalize` ends the pass: it clears
initialized and restores actuality.

**Notes** — the angle evaluator's setup additionally compares the reference vertex, since its
whole precomputed direction depends on it. That is the only parameter in the family whose
comparison is exact rather than approximate, and it must be: a vertex identifier is not a
measurement.
