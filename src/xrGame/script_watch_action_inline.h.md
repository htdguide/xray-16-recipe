# src/xrGame/script_watch_action_inline.h

> The constructors and setters of a look order: each fixes which of the four goal kinds the order is, and clears its completion flag.

**Needs** — [`script_watch_action.h`](script_watch_action.h.md) · [`sight_manager_space.h`](sight_manager_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field assignment with one invariant

## Purpose

Carries the bodies for [`script_watch_action.h`](script_watch_action.h.md). The only
decision in the file is which goal kind each construction path selects, and the rule that
*any* change to the order re-opens it.

## State

The order record declared in the header:

```text
RECORD WatchOrder
  completed      : bool          # inherited; starts true, so an untouched order is a no-op
  object_to_watch: optional<ClientObject>
  watch_type     : SightType     # default: keep the current direction
  goal_type      : GoalType      # default: current
  watch_vector   : vector3
  bone_to_watch  : text          # empty means the object's centre
  target_point   : vector3       # searchlight path only
  velocity_yaw   : real          # searchlight path only
  velocity_pitch : real          # searchlight path only
```

**Invariants** — every setter and every constructor clears `completed`. This is the file's
one load-bearing rule: an order is *finished* until something states a goal, and restating
a goal on a finished order restarts it. Without this a script that re-issues the same look
after it completed would be ignored forever.

Goal type is not set independently of the payload — it is a consequence of which
construction path ran:

```text
FROM sight type alone            -> goal = by_watch_type
FROM sight type + direction      -> goal = by_direction   (the direction setter sets it)
FROM sight type + object [+bone] -> goal = by_object      (the object setter sets it)
FROM target point + two speeds   -> goal = current        (searchlight; point is separate)
FROM object + two speeds         -> goal = by_object      (searchlight tracking a target)
DEFAULT                          -> goal = current, order already completed
```

**Notes** — the two-speed searchlight constructors explicitly zero the sight type and goal
type before filling their own fields; the creature path would otherwise inherit a
meaningless sight type. The searchlight reads the target point only when no object is set,
so the object field is the discriminator between "track this entity" and "sweep to this
point".

`initialize` is empty: a look order has nothing to do at the moment it is selected,
because every consumer re-reads the whole record each update.
