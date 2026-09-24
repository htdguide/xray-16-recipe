# src/xrGame/stalker_movement_params.cpp

> The movement-state record — what a stalker's locomotion should look like — and
> the lazy, time-throttled choice of which loophole of a smart cover it occupies.

**Needs** — [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — [`stalker_movement_params.h`](stalker_movement_params.h.md) · [`stalker_movement_params_inline.h`](stalker_movement_params_inline.h.md)
**Tier floor** — T2: a record with derived fields and a time-based cache.

## Purpose

One record holds everything about how a stalker should be moving: posture, gait, mental
state, which kind of path to plan, an optional exact destination, an optional exact facing,
and — the part that makes this more than a struct — which smart cover and which loophole
of it the creature occupies.

The movement manager keeps two of these, *current* and *target*, and locomotion is the
process of driving current toward target. Everything here exists to make that comparison
and that convergence correct.

## State

```text
RECORD MovementParams
  body_state        : enum {stand, crouch}            default stand
  movement_type     : enum {stand, walk, run}         default stand
  mental_state      : enum {free, danger, panic}      default danger
  path_type         : enum {no path, level path, game path, patrol, …}  default no path
  detail_path_type  : enum {smooth, …}                default smooth

  desired_position  : optional<Position>
  desired_direction : optional<Direction>             # invariant: unit length when set

  cover_id          : text = ""
  cover             : optional<SmartCover>            # derived from cover_id
  cover_loophole_id : text = ""
  cover_loophole    : optional<Loophole>              # derived; none means "let the
                                                      # record pick one for me"

  cover_fire_object   : optional<GameObjectRef>       # mutually exclusive with …
  cover_fire_position : optional<Position>            # … this one

  manager                 : MovementManagerRef        # the owner, for position_to_cover_from
  selected_loophole       : optional<Loophole>        # the lazily chosen aperture
  last_selection_time     : int                       # when it was chosen
  selected_loophole_valid : bool                      # whether it may be reused
```

Invariants:

- `cover` is exactly the cover named by `cover_id`, or absent when the identifier is
  empty. Setting the identifier is the only way to change it.
- `cover_loophole` is exactly the loophole named by `cover_loophole_id` within `cover`, or
  absent. Absent is meaningful: it means *no particular aperture was demanded*, and the
  record will select one itself.
- `cover_fire_object` and `cover_fire_position` are never both set; setting either clears
  the other.
- Setting `desired_position` or `desired_direction` clears `cover_id`. A stalker cannot be
  simultaneously headed for a free position and occupying a cover.
- The optional fields are stored as a value plus a presence marker rather than as a
  reference, so that a copy of the record is independent of the original — see the notes
  on assignment below.

## Assignment

**Contract** — copies every field, then re-establishes the three optional-field
presence markers to point at the *copy's* own storage rather than the source's. Does not
copy the owning manager: a copied record keeps whatever manager it was constructed with.
Preserves the lazy-selection cache, including its timestamp, so a copy does not force a
re-selection.

**Notes** — in the original, presence is encoded as a pointer that is either null or aimed
at a sibling field of the same object, which is why assignment has to fix it up. A rebuild
using a real optional type deletes this function entirely; what survives is the
requirement that *records are value-copied, not aliased*, and that the copy does not reset
the selection cache.

## `equal_to_target`

**Contract** — true when this record already describes the target state, meaning nothing
needs to be re-planned. Compares the five locomotion fields exactly, the two optional
vectors approximately, the cover identity and the fire target exactly, and the loophole
with one twist.

**Invariants** — the twist is the load-bearing line: when the *target* names no explicit
loophole, this record's loophole is compared against the target's lazily *selected*
loophole. Without that, a target that says "take this cover, any aperture" would never
compare equal to a current state that has already picked one, and the creature would
re-plan every frame forever.

```text
FUNCTION equal_to_target(target) -> bool
  IF detail_path_type <> target.detail_path_type        -> RETURN false
  IF path_type        <> target.path_type               -> RETURN false
  IF mental_state     <> target.mental_state            -> RETURN false
  IF movement_type    <> target.movement_type           -> RETURN false
  IF body_state       <> target.body_state              -> RETURN false
  IF desired_direction not approximately target's       -> RETURN false
  IF desired_position  not approximately target's       -> RETURN false
  IF cover_id         <> target.cover_id                -> RETURN false
  IF cover_fire_object<> target.cover_fire_object       -> RETURN false
  IF cover_fire_position not approximately target's     -> RETURN false
  IF target.cover_loophole IS none
    RETURN cover_loophole == target.selected_loophole   # the twist
  RETURN cover_loophole == target.cover_loophole
```

Note that the vector comparisons are on the stored values, not on presence: two records
that both have no desired position compare equal because both store the same sentinel.

## `cover_id`

**Contract** — sets the smart cover by name and resolves it through the cover registry.
A no-op when unchanged. Any real change clears the loophole (both the explicit one and the
lazily selected one), because a loophole name is only meaningful inside one cover. An
empty name means "no cover".

```text
FUNCTION cover_id(new_id)
  IF new_id == cover_id -> RETURN
  cover_id <- new_id
  cover_loophole_id("")            # drops the explicit loophole
  selected_loophole_valid <- false
  selected_loophole <- none
  cover <- (new_id is empty) ? none : cover_registry.smart_cover(new_id)
```

## `cover_loophole_id` (setter)

**Contract** — names the aperture within the current cover. Always clears both fire
targets first, even when the loophole is unchanged: choosing an aperture supersedes any
previously demanded firing target. Then, if the name actually changed, invalidates the
lazy selection and resolves the loophole against the cover's description, failing loudly
on an unknown name. An empty name restores "let the record pick".

## `actualize_loophole`

**Contract** — refreshes the lazily selected loophole if it is stale, asking the cover to
rank its own apertures against the position the creature is taking cover from. Mutates
cache fields only; callable from a read-only context.

Staleness has two causes and the ordering between them is the decision here:

```text
FUNCTION actualize_loophole()
  IF selection is marked valid
     AND (no cover, OR no selected loophole,
          OR the cover still contains the selected loophole)
     AND now < last_selection_time + 2000 ms
    RETURN                                  # still fresh
  selected_loophole_valid <- true
  last_selection_time <- now
  position <- manager.position_to_cover_from()
  selected_loophole <- cover.best_loophole(position,
                          allow_reselect = true,
                          prefer_current = (manager.current_params.cover == this.cover))
```

**Notes** — the two-second floor is the whole point of the cache. Ranking apertures is not
expensive, but doing it every frame makes a stalker's aim flicker between loopholes as its
enemy moves; the delay makes the choice visibly deliberate. Two seconds is not derivable
from anything else in the codebase — treat it as a tuned feel constant.

The second guard, "the cover still contains the selected loophole", catches the case where
the cover's description was reloaded underneath a live selection.

The `prefer_current` argument asks the cover to bias toward the aperture the creature is
already in, but only when the *current* movement state names the same cover — a target
state being evaluated for a cover the creature has not entered gets an unbiased ranking.

## `cover_loophole_id` (getter) and `cover_loophole`

**Contract** — both answer "which aperture is this record about". An explicitly named
loophole is returned directly; otherwise the lazy selection is refreshed and returned,
which may still be absent if the cover offers nothing suitable. Requires a cover to be
set. The getter returning a name and the getter returning the loophole itself differ only
in what they hand back.
