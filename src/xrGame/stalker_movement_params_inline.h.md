# src/xrGame/stalker_movement_params_inline.h

> The small setters of the movement-state record — the ones whose side effects
> are the interesting part.

**Needs** — [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_movement_params.cpp`](stalker_movement_params.cpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: field writes with invariant maintenance.

## Purpose

Split out of the header for compilation reasons only. The substance of the record is in
[`stalker_movement_params.cpp`](stalker_movement_params.cpp.md); what is genuinely here
is a small set of mutually-exclusive-field rules that are enforced nowhere else.

## State

None of its own.

## `construct`

**Contract** — binds the record to its owning movement manager, exactly once. A record
that has already been bound must not be rebound; a record must not be used unbound,
because loophole selection asks the manager what the creature is covering from.

## `desired_position` and `desired_direction` (setters)

**Contract** — set or clear an exact destination and an exact facing. Clearing writes a
sentinel into the stored value as well as dropping presence, so that the approximate
comparison in `equal_to_target` sees two cleared records as equal. Setting either one
**clears the cover identifier**: a stalker headed for a free position is not occupying a
smart cover, and the two states must never both be live.

A direction that is set must be unit length; this is asserted rather than normalized,
because a non-unit direction here means the caller computed it wrongly and the resulting
aim error would be silent.

## `cover_fire_object` and `cover_fire_position` (setters)

**Contract** — name what the cover is being taken against, as either an object or a fixed
position. Setting one clears the other, in both directions. Clearing a position also
writes the sentinel value; clearing an object does not touch the position.

Note the asymmetry: setting the object to nothing returns immediately and leaves the
position alone, whereas setting the position to nothing leaves the object alone too. So
"clear both" requires two calls, and callers that mean "cover against nothing in
particular" make them.

## Plain accessors

`desired_position`, `desired_direction`, `cover_id`, `cover`, `cover_fire_object` and
`cover_fire_position` as reads, with no behaviour.
