# src/xrGame/ai/monsters/states/state_move_around_point_inline.h

> The orbit state's implementation file: every statement in it is commented out, and no file includes it.

**Needs** — [`state_move_around_point.h`](state_move_around_point.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: nothing to implement

## Purpose

This file would implement the orbit-a-point leaf state. It implements nothing: the body of
its per-tick execution is entirely commented out and its completion test returns "not
finished" unconditionally. It is also unreachable — the state's declaration includes the
move-to-point implementation instead of this one, so even if a creature did select the
orbit state, these definitions would not be the ones found.

A rebuilder should delete it. It is recorded here because the mirror must be complete, and
because the disabled code says what the state was *meant* to do, which is worth one
sentence: set a movement target at the record's point and vertex, apply the record's
action, animation modifiers, acceleration profile and sound — exactly the move-to-point
state, with the orbit radius never consulted. Whatever made it an *orbit* was never
written.

## Exported units

None that function.
