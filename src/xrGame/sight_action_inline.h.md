# src/xrGame/sight_action_inline.h

> The six ways to state a look order, and the equality rule that decides whether a newly issued order is the one already running.

**Needs** — [`sight_action.h`](sight_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: construction and a comparison

## Purpose

Carries the bodies for [`sight_action.h`](sight_action.h.md). Two things here are
load-bearing: which sight type each construction path implies, and what "the same order"
means.

## Construction

**Contract** — six forms, each fixing the sight type and the torso-look flag:

```text
DEFAULT                                  -> current_direction
(type, torso_look?, path?)               -> that type; `path` selects the
                                            path-difference variant of the cover search
(type, direction_vector, torso_look?)    -> that type, payload the vector
(object, torso_look?, fire?, no_pitch?)  -> fire_object IF fire ELSE object
(perception_record, torso_look?)         -> object, payload a remembered perception record
(type, optional direction_vector)        -> that type, EXCEPT: fire_position is rewritten
                                            to position with torso_look forced on
```

**Invariants** — the rewrite in the last form is the only place `fire_position` is
handled, and it is why that sight type never reaches the execution switch. See
[`sight_manager_space.h`](sight_manager_space.h.md).

The `no_pitch` flag on the object form is a separate axis from torso look: it says to look
at the entity's bearing but keep the head level, which is what a creature does when the
target is far enough that tilting would look wrong.

## Equality

**Contract** — two orders are equal when they have the same sight type *and* the fields
that type actually uses agree. The comparison is per-type, and this matters: it decides
whether re-issuing an order restarts it.

```text
FUNCTION equals(other) -> bool
  IF sight_type != other.sight_type THEN RETURN false
  MATCH sight_type
    current_direction, path_direction -> torso_look agrees
    direction, position               -> torso_look agrees AND vectors are near-equal
    object, fire_object               -> torso_look agrees AND same entity
    cover, search                     -> torso_look agrees AND path flag agrees
    cover_look_over                   -> the glance interval agrees
    animation_direction               -> always equal
```

**Invariants** —

- Vector comparison is *approximate*, not exact. Two look-at-a-point orders one
  floating-point step apart are the same order, so a script recomputing a position every
  tick does not restart the look and reset its turn.
- `animation_direction` orders are always equal to each other: the payload is the
  animation, so there is nothing to compare, and restarting would fight the animation.
- The remembered perception record is *not* compared for object orders — only the entity
  is. Two orders to look at the same entity, one derived from memory and one from direct
  sight, are the same order.
- An unhandled sight type reaching this comparison is a programming error, not a false
  answer. That is deliberate: silently answering "different" would restart the order every
  tick and the head would never settle.

This method is what [`sight_manager.cpp`](sight_manager.cpp.md) uses to decide whether a
newly pushed order can be ignored.

## Accessors

**Contract** — the payload setters (direction/position vector, entity to look at,
perception record) write the field and nothing else — notably they do *not* change the
sight type or restart the order, unlike the script-facing look order in
[`script_watch_action_inline.h`](script_watch_action_inline.h.md). The getter for the
fire-at-object sub-state asserts the order really is a fire-at-object order, because the
field is meaningless otherwise.
