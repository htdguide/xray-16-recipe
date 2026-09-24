# src/xrGame/moving_object_inline.h

> The avoidance record's accessors, and the one rule that makes a decision's timestamp mean "since when", not "as of when".

**Needs** — [`moving_object.h`](moving_object.h.md)
**Used by** — [`moving_object.h`](moving_object.h.md)
**Tier floor** — T3: field access plus one state-change rule

## Purpose

Accessors for the avoidance record, separated out only because the original language wants
inline bodies after the class body; the split is arbitrary. The two action setters are the
exception and carry real behaviour.

## State

`Stateless` — reads and writes the record's fields.

## Setting the action

**Contract** — two forms, one with an associated position and one without. Both stamp the
current frame number unconditionally. Both then return early if the action is unchanged, so
that the *time* stamp and the action position are written only on an actual transition.

```text
FUNCTION set_action(new_action, optional position)
  action_frame := current frame            # always, even if nothing changes
  REQUIRE new_action IS move OR wait       # the other five values are never produced
  IF new_action == action THEN RETURN

  action := new_action
  action_position := position, or the infinite sentinel when none was given
  action_time := current wall-clock milliseconds
```

**Invariants** — the frame stamp and the time stamp answer different questions and must be
maintained differently. The frame stamp means *"this record was decided this frame"* and is
how the avoidance solver knows not to reconsider a creature twice in one pass; it must be
written on every call. The time stamp means *"the creature has been waiting since"* and is
how the solver applies its inertia, refusing to flip a decision that is only half a second
old; it must survive a re-assertion of the same action. Swapping the two behaviours breaks
either duplicate suppression or wait inertia.

Only *move* and *wait* are ever passed; the enumeration's four sidestep values and its
follow value are declared and never produced. They are the vocabulary of an avoidance
scheme that was planned and not built, and a rebuild may reduce the type to two states.

## Accessors

**Contract** — `id` gives the creature's name, `position` the indexed position, `radius` the
creature's radius, `object` the creature itself, and `action`, `action_position`,
`action_frame`, `action_time`, `static_query`, `dynamic_query` the corresponding fields.
The creature-dereferencing ones assert its presence, which is a construction invariant
rather than a runtime condition and so is checked only in checked builds.
