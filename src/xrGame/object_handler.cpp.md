# src/xrGame/object_handler.cpp

> The creature's hands: owns the object-handling planner, keeps it in step with the inventory, and answers where a weapon is attached and whether it is slung.

**Needs** — [`object_handler.h`](object_handler.h.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Torch.h`](Torch.h.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: planner ownership, bone lookup and inventory reconciliation

## Purpose

A stalker is an inventory owner that can *use* what it owns. This file is the layer between
those two facts: it owns the planner that decides how, forwards inventory changes into the
planner's item set so the planner always plans over what the creature actually has, and
answers the geometric and state questions the animation and render layers ask about the
weapon in the creature's hands.

The one substantial algorithm here is the sling-state question, which is subtler than it
looks because a weapon *in transition* between held and slung is neither.

## State

```text
RECORD ObjectHandler                      # mixed into a stalker, extends InventoryOwner
  planner            : ObjectHandlerPlanner
  hand_bone          : int                # right hand: where a held weapon hangs from
  left_finger_bone   : int
  right_finger_bone  : int
  strap_bone_0       : int                # cached, per weapon
  strap_bone_1       : int
  strap_object       : entity identifier  # which weapon the cached sling bones belong to
  hammer_is_clutched : bool
  clutched_hammer_enabled : bool
  infinite_ammo      : bool               # from the server record, at spawn
  inventory_actual   : bool               # cleared on every take and drop
```

**Invariants** — the sling bone pair is cached against the weapon's entity identifier, so it
is resolved once per weapon rather than per frame. A rebuild must invalidate it when the
weapon changes, which the identifier comparison does implicitly.

The three hand bones are resolved once at reinitialization from three configuration keys on
the *creature's* section, not the weapon's. Every creature model names its own weapon
attachment bones; every weapon assumes them.

## Lifecycle

**Contract** — construction allocates the planner. `reinit` clears the clutched-hammer state,
binds the planner to the creature, resolves the three hand bones from the creature's
configuration against its skeleton, and invalidates the cached sling bones. `net_Spawn`
defers to the base and then reads the infinite-ammunition flag out of the server record's
trader flags. Destruction releases the planner.

**Invariants** — the bone resolution must happen after the planner has been bound, because it
reaches the creature's model through the planner. That ordering is an artifact of the planner
holding the only back-reference at that point; a rebuild passing the creature in directly
removes the constraint.

The infinite-ammunition flag comes from the *server* record's trader flags, not from
configuration. It is per-entity authored state, so it survives a save and can be set by a
script on one particular creature.

## `OnItemTake`

**Contract** — when the creature gains an item: mark the inventory summary stale, add the item
to the planner's item set, switch on a torch if the creature is alive, clear the pending
item-to-spawn record if this item is the one that was owed, and if the item is a weapon,
install a recoil effector cloned from the weapon's camera recoil with the **AI** relaxation
speed substituted.

**Invariants** — the recoil is cloned rather than shared, and its relaxation speed is replaced
with a separate AI-specific value from the weapon's configuration. A creature must recover
from recoil differently than the player's camera does: the player's number is tuned for how
a view feels, the creature's for how quickly it can reacquire. Sharing one number makes
creature accuracy a function of camera feel.

## `OnItemDrop`

**Contract** — when the creature loses an item: mark the inventory summary stale; if the
creature has infinite ammunition, is alive, and the dropped item is *not* useful to a
creature, and it is an ammunition box, spawn a replacement box of the same kind at the
creature's position and record what was owed; then remove the item from the planner's item set
and switch off a torch.

**Invariants** — this is the other half of infinite ammunition. A creature discards an empty
box as useless; this intercepts that moment and immediately spawns a fresh one. The pending
record (what section, how many rounds) exists so that the take handler can recognize the
replacement's arrival and stop owing it, which is what stops a spawn storm if the spawn is
delayed by a frame.

The "not useful to a creature" test is what distinguishes an *empty* box from a full one being
dropped for another reason. A rebuild must keep the test, or a creature with infinite
ammunition will duplicate every box it hands over.

## `weapon_bones`

**Contract** — reports the three bones the currently active weapon should be attached to, and
sets the weapon's slung flag to match. Two answers:

```text
FUNCTION weapon_bones() -> (b0, b1, b2)
  weapon := the active item, if it is a weapon
  IF there is no weapon, OR the world state does not say "strapped"
    IF there is a weapon THEN clear its slung flag
    RETURN (hand bone, right finger bone, left finger bone)

  REQUIRE the weapon can be slung             # a hard failure, named in the message
  IF the weapon is not the one the sling bones are cached for
    resolve the weapon's two named sling bones against the creature's skeleton and cache them
  set the weapon's slung flag
  RETURN (strap bone 0, strap bone 1, strap bone 1)
```

**Invariants** — the third returned bone duplicates the second in the slung case. A held
weapon is attached at three points — the hand and two fingers — so that the model's grip
deforms with the animation; a slung one hangs from two, and the third slot must still be
filled with something valid.

The *weapon* names its sling bones and the *creature's skeleton* resolves them, so a weapon
that names a bone the creature does not have fails here. That is why the "can be slung"
requirement is a hard, named failure rather than a silent fallback: the alternative is a
weapon attached to bone zero, floating at the creature's origin.

**Notes** — this reader has a side effect — it sets the weapon's slung flag — because the
render layer asks for the bones and the weapon's own drawing needs the flag, and there is no
other moment guaranteed to happen for both. It is a wart; a rebuild should separate the query
from the update and drive the flag from the state change instead.

## `weapon_strapped` / `weapon_unstrapped`

**Contract** — the two sling questions, each answered for the active weapon or for a named
one. They are *not* negations of each other: during a transition both can answer false.

```text
FUNCTION weapon_strapped(weapon) -> bool
  IF the weapon cannot be slung at all THEN RETURN false
  IF the planner's current operator is any of the four sling operators
    almost := the operator is "strapping to idle"
    IF NOT almost OR the world state still says "strapped to idle" THEN RETURN false
  reconcile the weapon's slung flag with the world state
  RETURN the weapon's slung flag

FUNCTION weapon_unstrapped(weapon) -> bool
  IF the weapon cannot be slung at all THEN RETURN true     # it is always in hand
  IF the planner's current operator is any of the four sling operators
    almost := the operator is "unstrapping to idle"
    IF NOT almost OR the world state still says "strapped to idle" THEN RETURN false
  reconcile the weapon's slung flag with the world state
  RETURN NOT the weapon's slung flag
```

**Invariants** — the asymmetry in the no-sling case is deliberate and load-bearing: a weapon
that cannot be slung is *not slung* and *is unslung*, because it is permanently in hand.

Mid-transition, the answer is "no" to both unless the creature has reached the final leg of
the transition (the "to idle" operator) **and** the transition property has already been
cleared by that leg's animation-end callback. That combination is the earliest moment the
new state is true: the gesture is finished, only the return to neutral remains. Answering
earlier makes a weapon render at the wrong attachment point for a few frames; answering later
makes the creature appear to hesitate.

`weapon_unstrapped` with no active item answers **true**: a creature with empty hands has
nothing slung across its back that the plan needs to deal with.

**Notes** — `actualize_strap_mode` is the reconciliation step both questions call: it forces
the weapon's slung flag to agree with the world state's slung property, failing hard if the
state says slung and the weapon cannot be. It exists because the flag and the property can
drift — the flag is set by the bone query, which runs on the render path, and the property by
the planner, which runs on the update path.

## `is_weapon_going_to_be_strapped`

**Contract** — answers whether the planner's *target* state, not its current one, requires the
named object to end up slung. Looks up that object's slung-idle property in the target
state's sorted condition list.

**Invariants** — the target state's conditions are kept sorted, so the lookup is a binary
search. A rebuild storing them unsorted must not assume this.

**Notes** — this is a *future* question, and the only one on the page. It exists so that
something about to be drawn — an animation selector, a squad-mate's plan — can act on where
the creature is heading rather than where it is, without waiting for the transition.

## `set_goal` / `goal_reached`

**Contract** — `set_goal` forwards a named object action, a target object and four burst
parameters (minimum and maximum burst size, minimum and maximum interval between bursts,
both defaulting to 300 milliseconds) into the planner. An overload accepts an inventory item
and passes its underlying object. `goal_reached` answers whether the planner's current
solution is shorter than two steps.

**Invariants** — a solution of fewer than two steps means "the current state is the goal, or
one step from it"; the planner's solution always contains the current state as its first
element, so length one is arrival. A rebuild whose plan representation excludes the current
state must compare against one, not two.

The default burst interval of 300 milliseconds in both bounds means "no variation" — the
defaults produce a fixed cadence, and a caller wanting a varied one supplies both bounds.

## `aim_time`

**Contract** — the setter writes one inertia time into **six** planner operators at once: the
two aim operators, the two aiming-ready operators and the two force-full-aim operators, for
the named weapon. The getter reads it back from the first of them.

**Invariants** — all six must be set together. They are the six ways the plan can reach an
aimed state across the two weapon slots, and a creature whose aim time differs between them
aims for different lengths depending on which route the plan took — visible as inconsistent
reaction time.

**Notes** — the inertia time is the minimum an operator must run before the planner may
reconsider, so setting the aim time is literally "make this creature hold its aim this long".
It is per weapon and per creature, which is how a difficulty setting and a creature's rank
both reach the same number.

## `switch_torch` / `attach` / `detach`

**Contract** — attaching or detaching an item switches a torch on or off, but only if the item
really is an attached torch and the creature is alive. Both forward to the base afterwards
(attach) or beforehand (detach).

**Invariants** — the ordering differs between the two: attach delegates first then switches on;
detach switches off first then delegates. The switch needs the item to still be attached, and
detaching removes that, so the off-switch must precede it.

## `best_weapon`

**Contract** — asks the creature to refresh its best-item evaluation and returns the item it
chose to kill with. Returns nothing for a dead creature.

**Notes** — the evaluation itself lives on the creature, not here; this is only the entry
point. The refresh-then-read shape means the answer is always current as of the call, which
matters because the caller is usually the planner deciding what to draw.

## `update`

**Contract** — steps the planner. That is the whole per-frame contribution of this layer.

## `can_use_dynamic_lights`

**Contract** — answers from a global engine flag whether creature-held torches cast real
dynamic light.

**Notes** — a performance switch: a dozen creatures each casting a shadowing light is far more
expensive than a dozen glowing sprites. The flag is bit one of a shared common-flags word,
addressed by position rather than by name, which is a legibility cost with no benefit; a
rebuild should name it.
