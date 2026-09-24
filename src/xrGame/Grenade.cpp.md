# src/xrGame/Grenade.cpp

> A thrown grenade: an inventory item that is simultaneously a missile and an explosive, with the sleight of hand that the thing that leaves the hand is a second object and the thing in the inventory is destroyed.

**Needs** — [`Grenade.h`](Grenade.h.md) · [`Missile.h`](Missile.h.md) · [`Explosive.h`](Explosive.h.md) · [`Inventory.h`](Inventory.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`Entity.h`](Entity.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`Grenade.h`](Grenade.h.md); callers name that, not this file.
**Tier floor** — T2: state-machine transitions, timers and network events; the physics handoff is behind an interface

## Purpose

A grenade is two things at once — a **missile** (an item the first-person animation
throws) and an **explosive** (a thing with a fuse, a blast radius and a damage source) —
and this file is entirely about reconciling the two, plus one trick that a rebuilder
would otherwise have to rediscover.

**The trick**: the grenade the player holds is never the grenade that flies. Throwing
spawns a *fake missile* — a second, identical grenade object — arms it with the fuse and
the thrower's identity, launches it, and then destroys the held one and pulls the next
grenade of the same kind out of the rucksack into the hand. This is what makes a stack of
grenades behave like a stack: the item count is decremented by *destroying an item*, and
the flying object has an independent lifetime that outlives the throw animation.

Everything else here follows from that: the throw-end animation state is where the held
grenade dies, the replacement is chosen from the inventory at exactly that moment, and
the failure path (the player dies mid-throw, the item is force-deactivated) must still
release the primed grenade or the fuse would vanish with it.

## State

```text
RECORD Grenade
  remove_time        : int    # how long a grenade lying on the ground survives, milliseconds
  independency_time  : int    # server time it last became parentless; 0 while carried
  detonation_threshold : real # explosion damage above which a lying grenade cooks off
  thrown             : bool   # the held grenade has already launched its fake
  destroy_callback   : optional<callback>  # fires once, on death or detonation
  checkout_sound     : sound  # the pin-pull, classified as a weapon-recharge sound for AI hearing
```

Invariants:

- `thrown` is false whenever the grenade is in an inventory and true only between the
  launch and the end of the throw animation. It is reset when the *replacement* is
  slotted, not when the flying grenade lands.
- `independency_time` is non-zero exactly when the grenade has no parent. Becoming a
  child clears it and also clears the destroy time, because a carried grenade must never
  self-remove.
- The destroy callback fires at most once; it is cleared as it is called. Both the
  ordinary destruction path and the detonation path fire it, and whichever happens first
  consumes it.

## `Load`

**Contract** — reads the missile and explosive halves of the configuration, the pin-pull
sound, the ground-removal timeout (defaulting to thirty seconds) and the detonation
threshold (defaulting to a hundred damage points).

**Notes** — the pin-pull sound is registered with the AI-perception type of a weapon
being reloaded, not of an explosion. That is what makes a nearby creature react to a
grenade being primed the way it reacts to a rifle being cocked — a real gameplay decision
hidden in a sound-type constant.

## `net_Spawn`

**Contract** — spawn the client object, then derive the blast volume from the model's
own bounding box: take the largest of its three extents, make a cube of that size, and
scale it by three. Clears the thrown flag and the independency timer.

**Invariants** — the blast volume is a *cube* regardless of the model's shape. Using the
largest extent means a long thin grenade gets a blast box as wide as it is long.

**Notes** — the factor of three has no discoverable justification in the source. It is
the ratio between the grenade's physical size and the volume the explosive half searches
for victims in, so it is tuning; nothing else depends on the exact number.

## `Throw`

**Contract** — launch the fake missile. Does nothing if already thrown or if no fake
missile exists. Before delegating to the missile half, stamps the fake with the maximum
fuse time and with the thrower's entity identifier, and afterwards forces the fake into
the per-frame update set.

**Invariants** — the thrower's identity is set on the *fake*, not on the held grenade,
because the fake is what will deal damage and the kill must be attributed. The fuse is
set from the held grenade's configured maximum, so a grenade thrown without cooking gets
the full fuse.

```text
FUNCTION throw()
  IF thrown OR no fake missile THEN RETURN
  fake.fuse_deadline = configured maximum fuse
  fake.initiator     = the parent that is throwing      # for damage attribution
  missile_throw()                                       # imparts the velocity
  fake.enable_per_frame_update()
  thrown = true
```

**Notes** — the explicit activation of the fake's per-frame update is a bug fix, not a
design choice: the fake is spawned in a state where it is scheduled but not updated every
frame, and a flying grenade needs the frame-rate update for its fuse and its trail.

## `State`

**Contract** — reacts to two of the missile state machine's states. Entering *throw
start* plays the pin-pull sound at the item's centre. Entering *throw end*, **if the
grenade actually launched**, tears down its physics body, cancels its own destroy timer,
pulls the replacement grenade into the hand, and — on the authoritative side only —
destroys itself.

**Invariants** — the physics body is deactivated and released *before* the object is
destroyed, which is the preface's invariant about unreferencing an entity from the
physics world before releasing it. The destroy timer is set to "never" first so the two
destruction paths cannot race.

The replacement is pulled **before** the destruction request, so the inventory search
still sees this grenade and can exclude it.

## `PutNextToSlot`

**Contract** — after a throw, move the spent grenade to the rucksack and promote the next
grenade into the active slot. Runs only on the authoritative side. Looks first for
another grenade of the *same section*, then for any grenade in the grenade slot; failing
both, tells the actor to fall back to the previously held weapon.

**Invariants** — every inventory move made here is also announced as a network event, so
a client's inventory follows the server's. The search explicitly excludes this grenade,
and the result is asserted not to be this grenade — promoting the item being destroyed
would leave the player holding nothing.

```text
FUNCTION put_next_to_slot()
  IF this is a client mirror THEN RETURN
  move this grenade to the rucksack and announce it
  next = inventory.same_section(this, exclude_self)
  IF next is none THEN next = inventory.same_slot(grenade slot, this, exclude_self)
  IF next exists AND it fits its own base slot THEN
    place it, announce it, make its slot active
  ELSE
    tell the actor to return to the previous weapon slot
  thrown = false
```

**Notes** — preferring the same *section* before the same *slot* is what makes the game
hand the player another grenade of the same kind before switching kinds. It is a small
decision with a large effect on how the weapon wheel feels.

## `DropGrenade`

**Contract** — if the grenade is primed or mid-throw and has not launched, launch it and
report that it did. This is the "you were holding a live grenade when something happened"
path — used when the holder dies or is disarmed. Returns false for a grenade that is
merely being held.

## `DeactivateItem`

**Contract** — putting a grenade away while it is primed throws it instead. Stops the
current animation without letting its completion callback fire, then, on the
authoritative side and only if the item is not already being destroyed, throws.

**Invariants** — if the owner is *dead*, the throw force is zeroed and constant-power
throwing is disabled first, so a grenade released by a corpse drops at its feet rather
than being hurled. That is the difference between "he threw it" and "he dropped it", and
it is expressed entirely as two launch parameters.

## `DiscardState`

**Contract** — in single player only, a grenade in the ready or throwing state is snapped
back to idle. Used when the game needs the item to forget it was primed — the multiplayer
path must not do this, because the server has already been told.

## `SendHiddenItem`

**Contract** — the hide-this-item notification. If the grenade is mid-throw, throw first.
Then, for a grenade held by the player in the ready or throwing state, *suppress* the
notification entirely; otherwise pass it on.

**Notes** — suppression exists because hiding a primed grenade is not a legal transition:
the throw must complete and destroy the item, and a hide message arriving in between
would leave the server holding an item the client has already discarded.

## `Hit`

**Contract** — a grenade that takes *explosion* damage above its detonation threshold,
and that has no initiator yet, cooks off: it adopts the attacker as its initiator and
detonates. Any other damage passes through to the missile half.

**Invariants** — the initiator check is what stops a chain reaction from re-attributing a
grenade that is already detonating. Only a dormant grenade can be set off by a blast, and
the kill is credited to whoever caused the blast, not to the grenade's owner.

## `Destroy`

**Contract** — detonate. Fires and clears the destroy callback, finds the surface normal
under the grenade, and emits the explosion event at its position with that normal. The
normal is what orients the blast decal and the directional part of the effect.

## `NeedToDestroyObject` / `TimePassedAfterIndependant`

**Contract** — a grenade lying on the ground removes itself once it has been parentless
for longer than the configured removal time. Never in single player, and never on a
client mirror — only the authoritative side of a multiplayer game removes litter.

**Invariants** — the elapsed time is measured from *server* time, not local time, so the
lifetime is the same on every machine. A grenade with a parent reports zero elapsed time,
which is what keeps a carried grenade alive forever.

## `Useful`

**Contract** — whether the alife simulation should keep this grenade as a record. True
only for a grenade with no pending destruction, whose explosive half has nothing in
flight, and whose server flags permit saving. A grenade mid-flight or mid-detonation is
not saved, which is why a save taken at the wrong instant does not restore a hanging
explosion.

## `OnH_A_Independent` / `OnH_A_Chield` / `OnH_B_Independent`

**Contract** — the parentage transitions. Becoming independent stamps the server time, so
the ground-removal clock starts. Becoming a child clears that stamp and sets the destroy
time to "never", so picking a grenade up cancels its decay.

## `OnAnimationEnd`

**Contract** — the throw-end animation completing switches the item to hidden. Everything
else falls through to the missile half. The hidden state is what lets the replacement's
draw animation start.

## `Action`

**Contract** — after the missile half has had its chance, the "next weapon" command
cycles to the next grenade *kind* in the inventory rather than to the next weapon. The
switch is deferred rather than immediate, because it may need to interrupt an animation.

## `GetBriefInfo`

**Contract** — the HUD's item summary: short name, icon from the section, and — in place
of an ammunition count — the number of grenades of this section in the inventory. A
grenade stack's "ammo" is its own count.

## `UpdateCL`, `OnEvent`, `net_Destroy`, `net_Relcase`

**Contract** — each forwards to *both* bases, missile and explosive, in that order. The
per-frame update additionally runs position interpolation in multiplayer only, since in
single player the object's transform is authoritative locally.

**Notes** — the two-base forwarding is the whole cost of modelling a grenade as both a
missile and an explosive. A rebuild composing the two behaviours instead of inheriting
them gets the same effect with one dispatch, and must keep the order: the missile half
owns the transform the explosive half reads.

## `UsedAI_Locations`

**Contract** — whether the grenade occupies a navigation-graph position. Currently always
delegates to the missile half.

**Notes** — the source carries an unresolved note that this once returned a
state-dependent answer and crashed, because the object did *not* claim a navigation
position when it spawned but *did* claim one when it was destroyed, leaving the graph's
reference counts wrong. The lesson is general and worth carrying into a rebuild: whether
an entity occupies a navigation position must be a constant of the entity, not a function
of its state, unless both the spawn and the destroy paths consult the same answer.

## Notes

The default ground-removal time is thirty seconds and the default detonation threshold is
a hundred damage points. Both are overridable per section and neither has a derivation in
the source.
