# src/xrGame/ai/monsters/states/state_move_to_point_inline.h

> Implements both go-to-a-place states; the substance is in when each one decides it has arrived.

**Needs** — [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_data.h`](state_data.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`state_move_around_point.h`](state_move_around_point.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md)
**Tier floor** — T3: parameter forwarding plus two arrival tests

## Purpose

Both states do the same three things per tick — assert the action, hand the destination to
the path builder, engage the acceleration chain — and differ in what else they tell the
builder and in how carefully they decide they have arrived. Arrival is the interesting
part: the path builder's own "path ended" flag is not trusted on its own.

## `CStateMonsterMoveToPoint` — the plain form

**Contract** — entry clears the path builder. Each tick restates the destination (a point
plus an optional navigation vertex), restores the builder's generic parameters, and sets
the stop distance; optionally engages acceleration and a state sound. Completion is the
timeout, if any, **or** arrival.

```text
FUNCTION execute()
  object.set_action(data.action.action)
  object.animation.set_modifiers(data.action.spec_params)
  object.path.set_target_point(data.point, data.vertex)
  object.path.set_generic_parameters()
  object.path.set_distance_to_end(data.completion_dist)
  IF data.accelerated THEN engage acceleration profile and braking
  IF data.action.sound_type IS PRESENT THEN arm the state sound

FUNCTION check_completion() -> bool
  IF data.action.time_out != 0 AND expired THEN RETURN true

  # the builder's own end-of-path flag is necessary but not sufficient:
  # when the caller asked for an exact arrival (stop distance 0) the creature must
  # also actually be within one navigation cell of the point, measured in the
  # horizontal plane only, so that a destination on a ledge above or below does
  # not read as reached
  really_there = (data.completion_dist == 0)
                   ? horizontal_distance(data.point, object.position) < mesh_cell_size
                   : true
  RETURN object.path_builder.is_path_end(data.completion_dist) AND really_there
```

## `CStateMonsterMoveToPointEx` — the extended form

**Contract** — as above, plus: it sets the re-path cadence from the record, turns on cover
preference with fixed search parameters, and — when the record carries a non-degenerate
target direction — demands that the creature finish the move facing that way. It does
**not** restore the builder's generic parameters, because it is setting its own.

```text
FUNCTION execute()
  ... action, modifiers, target point as above ...
  object.path.set_rebuild_time(data.time_to_rebuild)
  object.path.set_distance_to_end(data.completion_dist)
  object.path.set_use_covers()
  object.path.set_cover_params(5, 30, 1, 30)   # hard-coded, see Notes

  IF magnitude(data.target_direction) > 0.0001
    object.path.set_use_dest_orient(true)
    object.path.set_dest_direction(data.target_direction)
  ELSE
    object.path.set_use_dest_orient(false)

  ... acceleration and sound as above ...

FUNCTION check_completion() -> bool
  IF data.action.time_out != 0 AND expired THEN RETURN true

  distance      = horizontal_distance(data.point, object.position)
  stop_distance = max(data.completion_dist, mesh_cell_size)

  # grace period: for the first 200 ms after entry, refuse to report arrival while
  # still outside the stop distance. This exists because the path builder reports
  # "path ended" for one tick before it has built anything, which would otherwise
  # make a freshly selected move state complete instantly and the behaviour above
  # thrash between two states.
  IF now() < time_state_started + 200 milliseconds AND distance > stop_distance
    RETURN false

  really_there = (data.completion_dist == 0)
                   ? distance < mesh_cell_size
                   : true
  RETURN object.path_builder.is_path_end(data.completion_dist) AND really_there
```

**Invariants**

- Arrival is judged **horizontally**. Vertical separation never prevents a creature from
  declaring it has arrived, which is what lets creatures finish a move onto a ramp or under
  an overhang, and is also why a destination directly above the creature reads as reached.
- The exactness threshold is the navigation mesh's cell size, not a tuned constant, so
  arrival precision scales with the level's mesh resolution rather than being fixed in
  world units.

## Notes

**The four cover numbers are hard-coded.** Every extended move asks the route search for
covered cells using the same 5 / 30 / 1 / 30 parameter set, regardless of creature or of
situation. A designer cannot tune this from configuration; a modder cannot either. If a
rebuild wants covered approach to differ between a cautious creature and a reckless one,
this is the line to lift into data.

**The 200 millisecond grace period** has no derivation in the source. It is large enough to
cover a route search issued on the frame of entry and small enough not to delay a genuinely
short move. That it is needed at all is a statement about the path builder's contract: its
end-of-path flag is not meaningful until it has been asked for a route.
