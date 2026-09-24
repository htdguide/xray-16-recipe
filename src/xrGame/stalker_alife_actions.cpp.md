# src/xrGame/stalker_alife_actions.cpp

> Two stalker behaviours for when nothing is happening: the idle stance a stalker holds on a level with no alife simulation, and the walk-over-and-take-it that picks up an item the stalker has noticed.

**Needs** — [`stalker_alife_actions.h`](stalker_alife_actions.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_base_action.h`](stalker_base_action.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`sound_player.h`](sound_player.h.md) · [`Inventory.h`](Inventory.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`movement_manager_space.h`](movement_manager_space.h.md) · [`detail_path_manager_space.h`](detail_path_manager_space.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — reached through its declarations in [`stalker_alife_actions.h`](stalker_alife_actions.h.md); callers name that, not this file.
**Tier floor** — T3: parameter assignment against the movement, sight and sound subsystems

## Purpose

Planner operators for a stalker with no threat, no order and nothing urgent. They are
implemented together because they share a shape: neither does any work of its own, both
*configure* the movement, sight, sound and item-handling subsystems and then let those run.
That is the shape of almost every operator in the stalker's behaviour set, and these two are
a good place to see it plainly.

## The operator lifecycle

Every operator here implements the same three-phase lifecycle, and **the order is
load-bearing**:

```text
initialize  — once, when the planner selects this operator.
              Set every subsystem parameter this behaviour depends on.
execute     — every brain cycle while the operator is selected.
              Set only what changes; re-assert nothing that initialize already set.
finalize    — once, when the planner selects a different operator.
              Undo only what would be wrong for an unknown successor.
```

Each phase calls the base's phase first. The division matters because operators are
*switched between* constantly: an operator that sets a parameter in `execute` that it could
have set in `initialize` costs that assignment every cycle, and an operator that fails to
undo a parameter in `finalize` leaks it into whatever runs next. Neither is caught by
anything; both show up as a stalker behaving slightly wrong for a while.

`finalize` is also where the "still alive" question is asked, because a dead stalker's sound
and animation subsystems have already been torn down and touching them is a fault.

## State

```text
RECORD ActionNoALife                        # extends the stalker operator base
  stop_weapon_handling_time : int           # world clock; when to put the weapon away

RECORD ActionGatherItems                    # extends the stalker operator base
  # no state; the item is read from the memory subsystem each cycle
```

## `CStalkerActionNoALife`

**Contract** — hold the idle stance used when the alife simulation is not running: wander
with no destination, walking, upright, relaxed, looking for cover, humming to itself, and
eventually stowing its weapon. Selected only when the alife-running property is false.

```text
FUNCTION initialize()
  base.initialize()
  desired position  = none            # no destination: wander
  desired direction = none
  path type         = game path       # coarse, cross-level: this stalker is not going anywhere in particular
  detail path type  = smooth
  body state        = standing
  movement type     = walk
  mental state      = free            # weapon may be stowed, gait is relaxed
  sight             = look for cover, not at anything

  stop_weapon_handling_time = now
  IF the stalker is currently holding its best weapon
    stop_weapon_handling_time = now + random in [30, 60) seconds

FUNCTION execute()
  base.execute()
  play the humming sound with a 60-second period and a 10-second spread
  IF now >= stop_weapon_handling_time
    goal = strap the best weapon, or plain idle if it has none
  ELSE
    goal = idle while holding the best weapon

FUNCTION finalize()
  base.finalize()
  desired position = none
  IF NOT alive  RETURN
  silence the sounds this behaviour started
```

**Invariants** — the randomized delay is only applied when the stalker is *already* holding
its best weapon. A stalker that arrives at this behaviour with the weapon stowed stows it
immediately; one that arrives holding it keeps holding it for half a minute to a minute
first. The effect is that a group of stalkers dropping into idle do not all sling their
weapons on the same frame, and that a stalker that has just finished a fight does not
disarm instantly.

The humming is started every cycle and the sound player deduplicates by period; the
explicit silencing in `finalize` is what stops it, because the sound outlives the operator
that started it.

**Notes** — the *game path* rather than a level path is the interesting choice. A game path
is a route over the coarse cross-level graph, which for a stalker with no destination
amounts to standing about; using a level path would have it pick a fine destination and walk
there. This behaviour is the one selected when there is no alife simulation to give stalkers
somewhere to be, so "stand about plausibly" is the whole requirement.

## `CStalkerActionGatherItems`

**Contract** — walk to the item the memory subsystem has selected as worth taking, looking
at it on the way. Re-enables pickup for that item and re-fires the take event on entry, so
an item previously given up on gets another chance.

```text
FUNCTION initialize()
  base.initialize()
  desired direction = none
  path type         = level path       # a specific destination on this level
  detail path type  = smooth
  body state        = standing
  movement type     = walk
  mental state      = danger           # weapon stays in hand while approaching loose gear
  silence the idle sounds
  goal = idle, holding whatever is already in hand

  selected = the memory subsystem's chosen item
  IF selected IS IN the ignored-touched set
    remove it from that set
    re-fire the take event for it       # gives the pickup another attempt

FUNCTION execute()
  base.execute()
  selected = the memory subsystem's chosen item
  IF selected IS none  RETURN           # it was taken, destroyed, or forgotten
  level destination vertex = selected.navigation vertex
  desired position         = selected.position
  sight                    = look at selected.position

FUNCTION finalize()
  base.finalize()
  sight            = look along the path
  desired position = none
  IF NOT alive  RETURN
  clear the sound mask
```

**Invariants** — the selected item is re-read every cycle rather than captured at
initialization, because it can vanish mid-approach: another stalker takes it, an explosion
destroys it, or the memory of it decays. Capturing it would leave the stalker walking to a
destroyed object.

The *danger* mental state while gathering, rather than free, keeps the weapon in hand. A
stalker crossing open ground to pick up loose gear is exactly when it is most exposed, and
the state also governs gait and posture.

**Notes** — the ignored-touched set is how a stalker stops retrying a pickup it could not
complete: touching an item it fails to take marks it ignored, so the memory subsystem stops
selecting it. Clearing that entry when this behaviour starts is the deliberate second
chance, and re-firing the take event is what actually attempts the transfer — walking to the
item does not pick it up, the touch event does.

The path-accessibility branch is commented out in the original: the destination vertex is
used directly rather than being replaced by the nearest accessible one if the stalker's
restrictions forbid it. A stalker restricted away from an item it can see will therefore
walk as close as its restriction allows and stop, rather than choosing a reachable
substitute. Whether that was a fix or a regression is not recoverable; the equivalent branch
*is* live in the smart-terrain operator, which suggests it was deliberate here.
