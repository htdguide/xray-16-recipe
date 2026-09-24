# src/xrGame/ai/stalker/ai_stalker_script_entity.cpp

> The adapters that let a Lua action queue drive a stalker directly, bypassing the planner — and the half of them that no longer does anything.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`script_entity_action.h`](../../script_entity_action.h.md) · [`object_handler_planner.h`](../../object_handler_planner.h.md) · [`sight_manager.h`](../../sight_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](../../stalker_movement_manager_smart_cover.h.md) · [`stalker_animation_manager.h`](../../stalker_animation_manager.h.md) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: adapters translating an authored action record onto four sub-managers

## Purpose

There are two ways to drive a stalker and they are mutually exclusive. The planner is one.
The other is the **script action queue**: a script pushes records describing a movement, a
watch target, an animation, an object interaction and a sound, and the stalker executes them
in order. While the queue has control, `Think` is not called at all — see
[`ai_stalker.cpp`](ai_stalker.cpp.md).

This file is the translation layer: four adapters that take one authored action record apart
and set it on the right sub-manager, plus four small weapon accessors the scripts use.

The finding a rebuilder needs: **the weapon-interaction adapter has been hollowed out.**
Its fire, reload and activate branches still exist and still set goals on the weapon-handling
planner, but every line that actually pulled a trigger, started a reload or moved an item
into a slot is commented out. What remains sets a *goal* and lets the weapon-handling planner
work out the rest — which is the correct architecture and leaves the branches looking like
they do more than they do.

## `bfAssignMovement` — the big one

**Contract** — takes one action record and configures four sub-managers from it, then
advances two of them immediately so the change takes effect this frame rather than next.
Returns whether the movement action is still running.

```text
FUNCTION assign_movement(action) -> bool
  IF the base adapter declines THEN RETURN false

  weapon_planner.set_goal(action.object.goal_type)   # what the hands should be doing

  movement.set_path_type(movement.current path type)  # see Notes: a self-assignment
  movement.set_detail_path_type(action.movement.path_type)
  movement.set_body_state(action.movement.body_state)
  movement.set_movement_type(action.movement.movement_type)
  movement.set_mental_state(action.animation.mental_state)
  sight.setup(action.watch.type, action.watch.vector)

  movement.update(this frame's delta)   # apply now, not next frame
  sight.update()
  RETURN true
```

**Invariants** — one record sets movement, posture, gait, mental state, gaze and weapon goal
together. That bundling is the script layer's whole vocabulary for "what is this character
doing", and it is why a scripted stalker looks coherent: a script cannot set a running gait
without also having stated a posture and a gaze.

**Notes** — the path-type line assigns the movement manager's current path type back to
itself. It is a no-op, and it is the one line in the bundle that does *not* come from the
action record — path type is chosen by the engine and the script only chooses the detail
path type within it.

## `bfAssignWatch` — where to look

**Contract** — four goal kinds, and a completion rule shared by three of them.

```text
FUNCTION assign_watch(action) -> bool
  IF the base adapter declines THEN RETURN false

  SELECT action.watch.goal_type
    object:
      IF no bone was named
        target = the object's centre
      ELSE
        # compose the named bone's local transform with the object's world
        # transform - this is how a script makes a stalker watch a specific
        # part of something, a vehicle's driver or a creature's head
        target = world position of that bone on that object
      sight.setup(action.watch.type, target)

    direction:
      sight.setup(action.watch.type, action.watch.vector)

    watch_type:
      nothing - the type alone was already applied

    current:
      # "keep looking where you are looking" completes immediately and reports
      # that the action is NOT still running
      action.watch.type = current_direction
      action.completed  = true
      RETURN false

  # for the three non-trivial kinds, the action is complete once the head has
  # actually converged on the target in both yaw and pitch
  IF goal_type is not watch_type
     AND head yaw is within epsilon of its target
     AND head pitch is within epsilon of its target
    action.completed = true
  ELSE
    action.completed = false

  RETURN NOT action.completed
```

**Invariants** — completion is measured on the *head's* convergence, not on a timer, so a
scripted look-at blocks the queue until the stalker has genuinely turned. The
`watch_type` kind is exempt because it changes only the gaze policy, not a target.

## `bfAssignObject` — what to do with an item

**Contract** — the weapon-interaction adapter. Nine goal kinds; most set a goal on the
weapon-handling planner and report whether that planner has reached it.

```text
FUNCTION assign_object(action) -> bool
  item = action.object.item

  IF the base adapter declines, or there is no item, or it is not an inventory item
    # "do nothing with anything": idle, holding whatever is already in hand
    weapon_planner.set_goal(idle, active item if any)
    action.completed = weapon_planner.goal_reached()
    RETURN NOT action.completed

  IF the item has no parent THEN RETURN true         # it is on the ground; nothing to do

  IF the active item is a magazine weapon
    set its burst length from the action's queue size

  SELECT action.object.goal_type
    idle:              set goal idle with the item; complete when reached
    fire_primary,
    fire_secondary:    set the corresponding fire goal with the item.
                       IF the weapon has no round chambered AND no reserve ammunition
                         mark the action complete - the script's fire order is
                         unsatisfiable and must not block the queue forever
    reload_primary,
    reload_secondary:  set the reload goal. IF the item named is not the active
                       item, log a script error and do nothing
    activate:          a TORCH is switched on directly; anything else becomes an
                       idle goal with that item
    deactivate:        a TORCH is switched off directly; anything else becomes a
                       bare idle goal
    use:               set the use goal; complete when reached
    take:              refuse if the item is already in the inventory, otherwise
                       route it through the ordinary pick-up reflex
    drop:              refuse if the item is not in the inventory, otherwise emit
                       an ownership rejection
```

**Invariants**

- Burst length is set from the action record on *every* call, before the goal switch, so a
  script that changes queue size between actions has it applied even to a fire order that
  then turns out to be unsatisfiable.
- The take path deliberately reuses the touch reflex rather than taking directly, so that a
  scripted take goes through the same authoritative round trip as an incidental one. A
  rebuild must not shortcut it.
- The torch is the only item type handled specially, because a torch has no weapon-handling
  goal — it is switched, not wielded.

**Notes** — the fire and reload branches are the hollowed-out ones. What remains is correct
as far as it goes: the goal is set and the weapon-handling planner does the work. The
commented-out lines would have driven the weapon's input commands directly, which duplicated
the planner. A rebuilder should implement the goal-setting form and ignore the residue.

## `bfAssignAnimation`

**Contract** — when the action names an animation, resets the torso and legs animation
channels so the scripted animation starts from a clean pose rather than blending out of
whatever gait was running. Returns whether the base adapter accepted the action.

## `GetCurrentWeapon` / `GetWeaponAmmo` / `GetMedikit` / `GetFood`

**Contract** — four accessors the script layer calls. The first two work: the active item as
a weapon, and the total suitable ammunition it could load.

**The last two always report nothing.** Both carry standing notes in the source saying they
should return the right item and do not. So a script asking a stalker for its medical kit or
its food gets an empty answer, every time, and any script logic built on them is dead. A
rebuilder should implement them — the inventory search is trivial — but must know that no
shipped script behaviour depends on them working, because none ever has.

## `ResetScriptData`

**Contract** — a pure delegation to the base adapter, present to complete the override chain.
