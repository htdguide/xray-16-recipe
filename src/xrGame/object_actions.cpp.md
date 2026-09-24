# src/xrGame/object_actions.cpp

> The concrete operators of the object-handling planner: eighteen small state machines that draw, sling, aim, reload, fire, throw and drop whatever a creature is holding.

**Needs** — [`object_actions.h`](object_actions.h.md) · [`object_actions_inline.h`](object_actions_inline.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`FoodItem.h`](FoodItem.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame operator steps driving an inventory and an animation manager

## Purpose

A creature does not manipulate its weapon directly; it issues the *same commands the player's
keyboard issues* into its own inventory, and lets the item's own state machine respond. This
file is the set of planner operators that do the issuing. That indirection is the central
design decision of the whole object-handling layer: one item implementation serves the player
and every creature, and an AI-held weapon reloads, jams and animates exactly as the player's
does.

Every operator here has the same three-point shape — set up once, step each frame, tear down
once — and the interesting content is which of the three each operator uses, because that is
where the timing bugs would be.

## State

Each operator holds only the item, the planner's property storage, and at most one extra
field. The stateful ones are noted individually.

## `CObjectActionCommand`

**Contract** — on setup, issues one raw inventory command (the same identifiers the player's
key bindings produce) with a press event. No per-frame step, no teardown.

**Notes** — the press is never matched with a release. Commands used this way are the
edge-triggered ones; the held ones (firing, zooming) have their own operators that pair the
two.

## `CObjectActionShow`

**Contract** — get the item into the creature's hands. On setup, if another item occupies the
item's slot, move that one to the rucksack and put this one in the slot. Each frame, if the
item is not already active: when nothing is in hand, put it in the slot and activate the slot;
when something is in hand and *is not mid-animation*, swap the slot occupant as at setup and
wait. Completes when the item is the active one.

```text
FUNCTION execute()
  IF the active item is this item THEN RETURN
  held := the active item
  IF nothing is held
    put this item in its slot; activate that slot; RETURN
  IF held is mid-animation THEN RETURN           # never interrupt a draw or stow
  IF this item's slot holds something else
    move that something to the rucksack
  put this item in its slot
```

**Invariants** — the mid-animation check is what makes weapon switching look right. Without
it, a creature whose plan changes twice in one second plays half of three draw animations.
Activating the slot is deliberately *not* repeated each frame — it is done once, on the frame
the creature's hands are empty, and the item's own state machine carries it from there.

## `CObjectActionHide`

**Contract** — each frame, activate the "no slot" pseudo-slot, which tells the inventory to put
whatever is held away, and clear the "has used enough" property. On teardown, if something is
still in hand and is not already hidden, interrupt its animation without firing the callback
and force it to idle.

**Invariants** — the teardown is the important half. A hide that is abandoned mid-animation —
because the plan changed — must leave the item in a defined state, and must not let the
interrupted stow animation report completion into the planner's world state. See
`stop_hiding_operation_if_any` in [`object_actions_inline.h`](object_actions_inline.h.md) for
the same treatment applied at setup by the sling operators.

## `CObjectActionReload`

**Contract** — on setup, assert the item is the active one, top up ammunition if the creature
has the infinite-ammunition property, and issue a reload press. Each frame, reissue the reload
press unless the weapon is mid-animation or is already as loaded as it can get.

```text
FUNCTION execute()
  IF the weapon is mid-animation THEN RETURN
  IF the magazine is not empty
    IF all suitable ammunition the creature carries equals what is in the magazine THEN RETURN
    IF the magazine is at capacity THEN RETURN
  issue reload
```

**Invariants** — the two guards together are what stop the loop. The first catches "I have no
spare ammunition to add"; the second catches "the magazine is already full". Without them a
creature with a full magazine reissues the reload command every frame forever, and the plan
never completes. The source's own comment names the symptom: repeated recharges.

**Notes** — the infinite-ammunition top-up (`try_advance_ammo`) scans the creature's belt and
then its rucksack for a box of any ammunition type the weapon accepts, and refills the **first
box that is not already full**, returning whether it refilled one. Belt before rucksack, and
one box per reload, not all of them — so a creature with infinite ammunition still consumes
boxes in the normal order and the inventory's other consumers see a consistent picture. This
is how the "unlimited ammunition" flag on a scripted creature is implemented; it is a refill,
not a bypass.

## `CObjectActionFire`

**Contract** — on setup and every frame, hold the trigger unless the creature would hit a squad
member, in which case release it. On teardown, always release.

```text
FUNCTION execute()
  IF the creature can kill a squad member right now
    release the trigger
    RETURN
  IF the weapon is not already in its firing state
    press the trigger
```

**Invariants** — friendly fire is checked *every frame*, not once at setup, because the thing
that changes is a squad member walking into the line of fire. Releasing the trigger rather
than aborting the operator keeps the plan intact: the creature stays aimed at its enemy and
resumes the instant the line is clear.

The re-press is conditional on the weapon not already firing, so the command is idempotent
across frames rather than a stream of presses.

**Notes** — the setup deliberately skips one level of its own inheritance chain and calls the
grandparent's setup. It thereby avoids the base's clearing of the aim-completed properties:
a creature that has finished aiming and is now firing must *keep* the aim recorded, or the
next frame's plan will decide it needs to aim again.

## `CObjectActionFireNoReload`

**Contract** — as the firing operator, but latches after the first burst. Holds one extra
field recording whether firing has been observed.

```text
FUNCTION initialize()
  grandparent setup                      # again, preserving the aim properties
  press or release the trigger by the friendly-fire test
  fired := the weapon is in its firing state

FUNCTION execute()
  IF fired
    IF the creature cannot hit a squad member THEN RETURN     # let the burst run out
    release the trigger                                        # a member walked in; stop now
    RETURN
  IF the creature can kill a squad member THEN RETURN
  IF the weapon is not firing THEN press the trigger
  IF the weapon is now firing THEN fired := true

FUNCTION finalize()
  release the trigger
  clear this operator's own world property
```

**Invariants** — the latch is never cleared once set within one activation of the operator; the
line that would clear it is commented out in the source. So the operator fires exactly one
burst per activation, and the planner must re-plan to fire again. That is the whole difference
from the ordinary firing operator, and it is what lets the planner interleave firing with
looking for cover.

Clearing the property on teardown rather than on completion means the effect does not persist
past the operator: the planner must observe the burst *while it happens*.

## `CObjectActionQueueWait`

**Contract** — wait for a burst already in flight to finish. On completion, and also on
teardown *if the operator did not complete* and the weapon is still the active item, tell the
weapon it was not stopped after its burst.

**Invariants** — the same release on both the success and the abandonment path is deliberate:
the latch it clears belongs to the weapon, not to the operator, and leaving it set would
prevent the weapon firing again regardless of what the planner decides next.

The teardown omits the "this item is active" assertions the other operators make, because
teardown can run after the item has been dropped, holstered or destroyed. Those assertions are
commented out in the source for exactly that reason.

## `CObjectActionAim`

**Contract** — aim the weapon; on the frame the operator completes, release the weapon's
burst-stop latch. Skips the base's aim-property clearing (grandparent setup again), since
clearing them is what this operator exists to undo.

**Notes** — the weapon reference is obtained by a narrowing conversion that may yield nothing
when the item is not a magazine-fed weapon; the check that would have rejected that is
commented out, so the operator tolerates aiming a non-firearm and simply skips the latch
release. A rebuild may make the type requirement explicit.

## `CObjectActionSwitch`

**Contract** — operate the firing-mode toggle. All three lifecycle points do nothing but
assert that the item is the active one and delegate.

**Notes** — the operator's *effect* is declared to the planner and applied by its base on
completion; the mode change itself is performed by the operator's precondition being met, not
by any command issued here. It is a pure marker operator. A rebuild may represent it as such.

## The four sling operators

`CObjectActionStrapping`, `CObjectActionStrappingToIdle`, `CObjectActionUnstrapping` and
`CObjectActionUnstrappingToIdle` share one shape and differ only in which property their
animation-end callback writes.

**Contract** — on setup: interrupt any stow animation in progress, optionally pre-set a
transition property, and register a callback on the creature's **torso** animation channel. On
each frame: nothing but the dead state-switch guard. On teardown and on destruction: remove the
callback if it is still registered.

```text
FUNCTION initialize()
  REQUIRE this item is the active one
  callback_registered := true
  stop_hiding_operation_if_any()
  set the operator's transition property       # only two of the four do this
  register on_animation_end on the torso channel

FUNCTION on_animation_end()
  set the operator's outcome property
  remove the callback
  callback_registered := false

FUNCTION finalize() / destructor
  IF callback_registered THEN remove the callback; callback_registered := false
  ELSE REQUIRE the callback is genuinely absent
```

The four assignments:

```text
  strapping             sets "strapped_to_idle" at setup, "strapped" at animation end
  strapping_to_idle     sets nothing at setup,            clears "strapped_to_idle" at end
  unstrapping           sets "strapped_to_idle" at setup, clears "strapped" at end
  unstrapping_to_idle   sets nothing at setup,            clears "strapped_to_idle" at end
```

**Invariants** — the transition property is set at setup and the outcome property at the
animation's end, which is what makes the transition atomic to the planner: while the animation
plays, the world state says "in transition", so no other operator's preconditions are met and
the planner cannot interleave anything.

The registration flag must be maintained on three paths — animation end, teardown and
destruction — and each path asserts the other two left things consistent. This is the
engine-wide "a destroyed object is unreferenced before its memory is released" invariant in
miniature: a callback outliving its operator is a call into freed memory the next time the
torso animation ends.

The callback goes on the **torso** channel specifically. A creature slinging a rifle keeps
walking, so the legs must be free; the torso channel is the one whose completion means the
slinging gesture is done.

**Notes** — the two "to idle" variants exist because the sling transition is not symmetric in
the animation data: going from slung to held and from held to slung each need a separate
return-to-neutral leg, which the planner sequences explicitly rather than folding into the
main leg.

## `CObjectActionDrop`

**Contract** — on setup, if the item exists and the creature really is its owner, send an
ownership-rejection event naming the item's entity identifier. Nothing else.

**Invariants** — the transfer goes through the **event** path rather than a direct inventory
call, because ownership is authoritative state: the server object must record the change, and
in a networked game the client cannot simply decide it. This is the one operator on this page
that crosses from the client side of the entity to the server side.

The ownership check before sending is not defensive noise — the planner can reach this
operator for an item the creature has already lost, and sending the rejection anyway would
make another creature drop the item.

## `CObjectActionIdle`

**Contract** — on setup, if the world state says the creature has used the item enough, order
the object handler to activate it as a goal; then clear that property.

**Notes** — this is how a creature finishes eating or using a medical item: the idle operator
is where the "I am done using this" state is noticed and converted into an activation order,
because idle is the state the plan necessarily passes through afterwards.

## `CObjectActionIdleMissile`

**Contract** — on setup, clear three of this item's properties: throw-started, throw-idle and
firing.

**Invariants** — the three are cleared through the planner's per-item property minting, not by
name, so they name *this grenade's* state and not another's. Returning a thrown-object plan to
idle must reset the whole throw sub-plan, or the next throw resumes mid-gesture.

## `CObjectActionThrowMissile`

**Contract** — on setup, press the zoom command (which for a thrown object begins the
wind-up), and set the operator's inertia time from the distance to the throw target. On
completion, release the zoom command, which is what actually lets go.

```text
distance := distance from the creature to its throw target
inertia_time := 2500 ms  IF distance > 45
              | 2000 ms  IF distance > 30
              | 1500 ms  IF distance > 15
              | 1000 ms  otherwise
```

**Invariants** — the inertia time is the minimum time the planner must leave this operator
running before it may reconsider. Holding the wind-up longer throws further, so the schedule
is a crude ballistics table expressed as "how long to hold": four bands over the useful range
of a thrown object. The thresholds are in metres and the times in milliseconds, and neither
the exact bands nor the exact times are derivable from anything else in the source — they are
tuned against the shipped throw animation and the grenade's launch speed. A rebuild changing
either must retune them, and a rebuild reproducing the original's grenade accuracy must copy
them as written.

Releasing on *completion* rather than at teardown matters: a throw abandoned mid-plan keeps
the grenade in hand rather than lobbing it somewhere unintended.
