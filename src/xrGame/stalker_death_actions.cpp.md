# src/xrGame/stalker_death_actions.cpp

> The dying reflex: a stalker shot mid-burst keeps firing as it falls, then drops what it was holding and becomes lootable.

**Needs** — [`stalker_death_actions.h`](stalker_death_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md)
**Used by** — [`stalker_death_actions.h`](stalker_death_actions.h.md)
**Tier floor** — T2: drives the inventory and the weapon's own input path.

## Purpose

Two jobs, in order. First, the death spasm: if the creature died with a finger on the
trigger it empties the magazine into wherever the body is pointing. Second, the transition
to a corpse: every carried item is moved out of the equipped slots into the backpack, the
weapon in hand is dropped where the body falls, and the creature is declared finished so
the brain can stop.

The order is the whole design. Dropping first would cancel the spasm; declaring the
creature finished first would end the action before either happened.

## State

`Stateless.`

## `fire` (private predicate)

**Contract** — should the body keep shooting. Pure; reads inventory, weapon and physics
state.

```text
FUNCTION should_keep_firing() -> bool
  IF carrying nothing                   THEN RETURN false
  IF the item in hand is not a weapon   THEN RETURN false
  IF the weapon's magazine is empty     THEN RETURN false
  IF the creature's trigger finger was not clutched THEN RETURN false
  IF the body cannot drop its weapon    THEN RETURN true       # see Notes
  RETURN time_since_death <= 0.5 seconds
```

**Invariants** — the clutched-trigger test is what makes this a *reflex* rather than a
flourish. A creature killed between bursts drops silently; only one killed mid-burst
sprays. That single condition is why the behaviour reads as physical rather than scripted.

**Notes** — the half-second window bounds the spasm in the ordinary case. The branch above
it is the exception: when the body's physics state forbids dropping the active weapon — the
creature died inside a smart cover or in a pose where a dropped rifle would fall through
geometry — the firing continues without the time limit, because the alternative is a body
holding a weapon it can neither drop nor discharge. This is an unusual shape: a time limit
that is skipped precisely when the tidy-up path is unavailable.

## `initialize`

**Contract** — start the spasm and strip everything that is not the weapon in hand.

```text
FUNCTION initialize()
  base.initialize()
  IF the creature is already being destroyed THEN RETURN
  IF NOT should_keep_firing() THEN RETURN

  press and hold the weapon's fire input
  IF the active slot is the rifle slot THEN
    set the weapon's burst length to its entire remaining magazine
  FOR EACH equipped slot
    skip the bolt slot
    skip the active slot
    move the item in it into the backpack
```

**Invariants** — the burst length is set to the whole magazine rather than the creature's
tuned burst shape, because a dead man does not release the trigger. Only the rifle slot
gets this: pistols and other slots keep their own firing rules.

The bolt slot is skipped everywhere in this file. Bolts are an infinite probing tool rather
than loot, and moving them around a corpse's inventory serves nothing.

**Notes** — the fire input is *pressed*, in the same way the player's input would press it,
rather than issued as a weapon goal. That is deliberate: the goal system belongs to a living
creature's action planner, and the planner is about to stop running.

## `execute`

**Contract** — per cycle: stop the body walking, keep firing while the reflex lasts, and
once it is over drop everything and declare the creature completely dead.

```text
FUNCTION execute()
  base.execute()
  IF the creature is already being destroyed THEN RETURN
  movement.enabled := false
  IF should_keep_firing() THEN RETURN                 # still spasming
  IF the body cannot drop its active weapon THEN RETURN   # wait for a pose that allows it

  FOR EACH equipped slot
    skip the bolt slot
    IF it is the active slot THEN mark that item to be dropped where the body lies
    ELSE move the item into the backpack
  set property Dead = true
```

**Invariants** — movement is disabled on every cycle, not once. The movement layer can be
re-enabled from elsewhere — a ragdoll settling, a script — and a corpse that starts walking
is the failure this guards against.

The property is set only after the drop has actually happened, which is what makes the
"cannot drop yet" early return safe: the action simply runs again next cycle.

**Notes** — the distinction between *dropping* and *stowing* is what makes a corpse lootable
in the way the game expects. The weapon that was in the creature's hands lands on the ground
next to it, visible; everything else ends up in the backpack, which is what the player opens.
A rebuild that drops everything leaves a pile of items around each body; one that stows
everything loses the visual cue that tells a player where a firefight happened.
