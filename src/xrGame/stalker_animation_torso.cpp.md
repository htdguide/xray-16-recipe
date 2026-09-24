# src/xrGame/stalker_animation_torso.cpp

> Chooses the motion a stalker's upper body plays, from what it is holding, what that item is doing, how the body is postured and how it is moving.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`stalker_animation_state.h`](stalker_animation_state.h.md) · [`stalker_animation_names.h`](stalker_animation_names.h.md) · [`stalker_animation_pair.h`](stalker_animation_pair.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`Missile.h`](Missile.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a decision tree over indices into the preloaded motion table, run once per creature per frame.

## Purpose

The torso is the body part that shows what a stalker is *doing*, and it is the only part
whose animation depends on three independent things at once: the item in hand and that
item's own state machine, the creature's posture and gait, and — for one narrow case — a
plan-level intention. This file is the resulting decision tree.

It is deliberately nothing but indexing. Every name was resolved to a motion handle at
model load (see [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md)), so this
runs every frame for every visible stalker and must not touch a string. The consequence is
that the whole file addresses animations by number, and the numbers only mean anything
against the fragment tables in
[`stalker_animation_names.cpp`](stalker_animation_names.cpp.md). That indirection is the
main thing a rebuild should fix: give the weapon-action indices names and this file becomes
readable.

## State

No state of its own; it reads the animation manager's. Three of the manager's fields are
consulted or written here:

```text
special_danger_move   : bool   # this creature is tuned to carry its weapon in the "danger" posture
looking_back          : int    # 0 = not glancing back, 1 = over the left, 2 = over the right
change_direction_time : int    # earliest time the creature may glance back again
```

**Invariants** — `looking_back` is non-zero only while a look-back motion is running. It is
cleared by the torso's end-of-motion notification, and clearing it also arms a cooldown, so
a stalker cannot glance over its shoulder on consecutive motions.

## `assign_torso_animation`

**Contract** — the entry point: returns the motion the torso should be playing this frame.
Pure with respect to the world; reads the creature's inventory, its posture and its item
state machines. Never fails — every branch ends at a motion handle, which may be invalid if
the model does not ship it.

```text
FUNCTION assign_torso_animation() -> Motion
  posture := current body state          # crouched, normal, or damaged-normal
  IF no active inventory item THEN
    RETURN empty_handed_animation(posture)

  classify the active item                # sets weapon / missile / neither

  IF holding a weapon THEN
    IF the weapon is strapped to the back THEN RETURN empty_handed_animation(posture)
    RETURN weapon_animation(item.animation_slot, posture)

  IF holding a throwable THEN RETURN missile_animation(item.animation_slot, posture)

  RETURN generic_item_animation(item.animation_slot, posture)
```

**Invariants** — a *strapped* weapon animates exactly like empty hands. That is why the
strapped test comes before the weapon branch rather than inside it: a slung rifle is not a
held rifle, and the whole weapon state machine is irrelevant while it is on the back.

## `aim_animation`

**Contract** — picks within the aim family, redirecting to the *special danger move*
variants when the creature is tuned for them and the conditions hold. Pure.

```text
FUNCTION aim_animation(slot, actions, index) -> Motion
  # index is the ordinary aim sub-entry: 0 standing, 2 walking, 3 running
  IF NOT special_danger_move      THEN RETURN actions.aim[index]
  IF slot is not the rifle slot   THEN RETURN actions.aim[index]
  IF the model ships fewer than 7 aim sub-entries THEN RETURN actions.aim[index]
  RETURN actions.aim[ index mapped: standing -> 4, walking -> 5, running -> 6 ]
```

**Invariants** — the mapping is total over exactly the three indices the callers pass. Any
other index is a programming error and is treated as unreachable rather than defaulted,
because silently returning the wrong posture would be invisible in testing.

**Notes** — three guards, three different kinds of reason. The first is per-creature tuning
(most stalker types are configured to use the danger carry, a few are not). The second is
that the variants are authored only for the rifle slot; a pistol has no danger carry. The
third is compatibility with models that predate the variants: a shorter sub-list means the
model does not have them, and the creature silently falls back to the ordinary aim. That
third guard is the one a rebuild will be tempted to drop, and it is what keeps the engine
running on unmodified retail data and on mod models alike.

## `no_object_animation`

**Contract** — the motion for a stalker with nothing in its hands. Branches first on
*mental state*: a creature at ease uses a relaxed idle/walk/run set; a creature in any
alerted state uses the aim family even with no weapon, because the aim family is what
carries the wary posture.

```text
FUNCTION empty_handed_animation(posture) -> Motion
  actions := torso_actions(posture, slot = 0)     # slot 0 is the no-item slot
  IF mental_state is at-ease THEN
    # at-ease is only reachable standing; crouching while relaxed is a contradiction
    REQUIRE posture is standing
    IF the creature is not moving THEN RETURN actions.idle[relaxed]
    RETURN actions[walk or run, chosen by gait][relaxed]

  IF the creature is not moving THEN RETURN aim_animation(0, actions, standing)
  IF gait is walk               THEN RETURN aim_animation(0, actions, walking)
  RETURN aim_animation(0, actions, running)
```

**Invariants** — the walk and run entries are addressed as *one base index plus the gait
value*, which requires the two fragment table entries to be adjacent and ordered like the
gait enumeration. The coupling is invisible from both sides; see
[`stalker_animation_names.cpp`](stalker_animation_names.cpp.md).

## `unknown_object_animation`

**Contract** — the motion for a stalker holding something the animation layer has no
special handling for (a detector, a quest item, an artifact). Branches on what the *item
handling planner* is currently doing rather than on the item, because a generic item has no
state machine of its own the torso could read.

```text
FUNCTION generic_item_animation(slot, posture) -> Motion
  actions       := torso_actions(posture, slot)
  standing_set  := torso_actions(standing, slot)     # note: always the standing set
  CASE item_planner.current_action
    firing, aiming, getting ready, waiting between bursts:
      RETURN moving_aim_or_lookback(slot, posture, actions)
    strapping         : RETURN standing_set.strap[to_strapped]
    unstrapping       : RETURN standing_set.unstrap[to_unstrapped]
    strapping_to_idle : RETURN standing_set.strap[to_idle]
    unstrapping_to_idle: RETURN standing_set.unstrap[to_idle]
  # nothing item-specific is happening: same shape as empty hands
  RETURN the mental-state branch from empty_handed_animation, using this slot
```

**Invariants** — the four slinging transitions always come from the **standing** set,
whatever the creature's posture. A stalker slinging a rifle while crouched plays the
standing transition, because the transitions are authored once. This is a data limitation
promoted to a rule, and a rebuild that authors crouched variants can drop the special case.

**Notes** — `moving_aim_or_lookback` is the shared tail also used by the firing branch of
the weapon selector:

```text
FUNCTION moving_aim_or_lookback(slot, posture, actions) -> Motion
  IF not moving THEN RETURN aim_animation(slot, actions, standing)
  IF posture is standing AND slot is the rifle slot AND may_look_back() THEN
    RETURN actions[look_back_left or look_back_right, per looking_back][gait]
  IF gait is walk THEN RETURN aim_animation(slot, actions, walking)
  RETURN aim_animation(slot, actions, running)
```

The look-back animations are the reason this tail exists. They are strafing-backwards
motions with the head turned, played when a stalker retreats while keeping a weapon on a
threat, and they are authored only for a standing creature with a rifle. Which shoulder it
glances over is chosen elsewhere and held in `looking_back`; this selector only consumes it.

## `weapon_animation`

**Contract** — the motion for a stalker holding a firearm, branching on the weapon's own
state machine. The weapon, not the creature, is authoritative here: what the torso plays
while reloading is decided by which third of the reload the weapon is in.

```text
FUNCTION weapon_animation(slot, posture) -> Motion
  actions := torso_actions(posture, slot)
  CASE weapon.state
    reloading:
      CASE weapon.reload_substate
        begin       : RETURN actions.reload[0]
        in_process  : RETURN actions.reload[1]
        end         : RETURN actions.reload[2]
    drawing   : RETURN torso_channel.select(actions.draw)      # variant list, sticky
    holstering: RETURN torso_channel.select(actions.holster)
    holstered : RETURN empty_handed_animation(posture)
    firing (either trigger):
      IF not moving THEN RETURN actions.attack[standing]
      IF posture is standing AND slot is the rifle slot AND may_look_back() THEN
        RETURN actions[look_back_left or right][gait]
      IF gait is walk THEN RETURN actions.attack[standing]     # see Notes
      RETURN actions.attack[running]
  # any state the torso has no opinion about
  RETURN generic_item_animation(slot, posture)
```

**Invariants** — the reload substates must be exhaustive. A weapon that reports a reload
substate the torso does not know about would otherwise fall through to an undefined motion,
so the case is treated as unreachable rather than defaulted.

**Notes** — the draw and holster branches go through the animation channel's *sticky
variant selector* rather than indexing: those two have several authored variants and the
selector must return the same one for the whole duration of the draw, not reroll each
frame. Every other branch indexes directly because it has one authored motion.

A stalker *walking while firing* plays the standing fire motion, not a walking one. The
original marks the two sub-indices with a comment noting what they "should" be, and uses
the standing entry anyway. The most likely reading is that the walking-fire motions were
never authored convincingly; it may equally be a bug that shipped. Either way it is visible
in the game and a faithful rebuild reproduces it.

The reload begin/in-process/end split is what lets a reload be interrupted: the creature can
leave the reload at a substate boundary and the torso follows, instead of being locked into
one long motion.

## `missile_animation`

**Contract** — the motion for a stalker holding a throwable, branching on the throwable's
state machine. The throw is a four-phase sequence and each phase is its own motion, which is
what allows a stalker to hold a primed grenade indefinitely before throwing it.

```text
FUNCTION missile_animation(slot, posture) -> Motion
  actions := torso_actions(posture, slot)
  CASE missile.state
    drawing     : RETURN torso_channel.select(actions.draw)
    holstering  : RETURN torso_channel.select(actions.holster)
    throw_start : RETURN actions.attack[0]        # wind up
    ready       : RETURN actions.attack[1]        # primed, held
    throw       : RETURN actions.attack[2]        # release
    throw_end   : RETURN actions.aim[0]           # recover to idle
    bore        : RETURN actions.attack[1]        # idle fiddling reuses the held pose
    holstered   : RETURN actions.aim[0]
    idle, or anything else:
      IF not moving  THEN RETURN actions.aim[0]
      IF gait is walk THEN RETURN actions.aim[2]
      RETURN actions.aim[3]
```

**Notes** — unlike the weapon selector, the fall-through here is the *idle* case rather
than a delegation to the generic selector. A throwable always has a defined pose, so there
is nothing to delegate to.

In diagnostic builds every branch first checks that the model actually ships the sub-entry
it is about to index and names the offending visual if not. That is the file's answer to
its own design: because everything is an index, a model missing one motion produces a
nonsense pose or a crash somewhere far away, and these checks pull the report back to the
model that caused it. A rebuild that keeps names or optional handles does not need them.

## `torso_play_callback`

**Contract** — invoked by the renderer when a torso motion finishes. Notifies the torso
channel's own subscribers, and — if the motion that just ended was a look-back — clears the
look-back state and starts a cooldown before the creature may glance back again.

```text
FUNCTION on_torso_motion_finished(blend)
  creature := blend.owner
  creature.animation.torso_channel.on_animation_end()
  IF creature.animation.looking_back != 0 THEN
    creature.animation.change_direction_time := now + LOOK_BACK_COOLDOWN
    creature.animation.looking_back          := 0
```

**Notes** — the cooldown is two seconds. It is not a timing detail of the animation; it is
what stops a retreating stalker from glancing over its shoulder every time the motion loops,
which reads as a nervous tic. The same field gates the creature's direction changes, so the
cooldown also means a stalker that just looked back commits to its current heading for that
long.
