# src/xrGame/Actor_Events.cpp

> Every change to the actor that must be authoritative arrives here as a message: taking and dropping items, eating, equipping, entering a holder, being teleported, being healed.

**Needs** — [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`Level.h`](Level.h.md) · [`Artefact.h`](Artefact.h.md) · [`Weapon.h`](Weapon.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: decodes a bit-packed message and mutates world state

## Purpose

This file is where the chapter's central architectural claim becomes concrete: **nothing
that changes the world happens by calling a method.** Picking up a rifle, eating a
bandage, sitting in a car, being teleported by a script — each is a message with an
identifier, decoded here and applied. In single player the message is generated and
consumed in the same process a moment later; in multiplayer it crosses the wire. One code
path either way.

Reading this file is the fastest way to learn what the actor's authoritative surface
actually is: the list of cases *is* the list of things that can happen to a player.

## State

`Stateless.` Every case mutates the actor, the inventory or the condition; nothing is
owned here.

## `OnEvent`

**Contract** — decodes one message and applies it. The base entity and the inventory-owner
mix-in are both given the message first, unconditionally, before the actor's own switch
runs — so a message may be handled at several levels. Unknown messages fall through
silently.

**Invariants**

- **Every entity reference on the wire is an identifier, resolved through the level's
  object registry.** A resolution failure is logged and the message dropped, never
  asserted in a shipping build: a message can legitimately name an object that has since
  been destroyed, and dying over it would make a lagging client unplayable.
- **A dead player in multiplayer may not take, use or act.** The check appears in three
  separate cases rather than once, because it is *not* applied in single player, where a
  dying actor must still be allowed to complete an in-flight action.

### take and buy

```text
resolve the object; IF missing THEN log and drop
IF multiplayer AND dead THEN drop
IF the inventory will accept it THEN
  reparent the object to the actor
  put it in the inventory (not forced, with a network notification)
  reconsider which weapon should be active
ELSE
  IN SINGLE PLAYER: send a REJECT message back at ourselves
  IN MULTIPLAYER: log an error and do nothing
```

**Notes** — the refusal path differs between the two modes and the difference is
load-bearing. In single player a refused take *becomes a drop*, because the item has
usually already been detached from the world by whoever sent the take; in multiplayer the
server owns that decision and the client must not invent one.

### reject and sell

```text
resolve the object; IF missing THEN log and drop
read the optional "about to be destroyed" flag         # absent means false
suppress creating a physics shell when selling or when about to be destroyed
REQUIRE the object's parent is this actor              # checked, then logged and dropped
IF the object is not already being destroyed AND the inventory releases it THEN
  deny the actor's touch sense this object for one second
  IF the message carries a position THEN move the object there
IF not about to be destroyed THEN reconsider the active weapon
```

**Invariants** — the touch-sense denial is what stops a dropped item from being instantly
picked up again by the pickup logic that is still running. One second is long enough for
the player to release the key and short enough not to be noticed.

**Notes** — the message has **three optional trailing fields** and is parsed by testing for
end-of-stream. That is a forward-compatible extension mechanism grown by accretion; a
rebuild defining its own protocol should version the message instead.

Selling suppresses the physics shell because a sold item goes to a trader's inventory and
must never exist in the world; an item about to be destroyed suppresses it because
creating a rigid body and immediately freeing it is pure cost.

### inventory action

**Contract** — a remote player's key press, replayed locally. Carries the action, its phase,
and **two random seeds** — one for aim sway and one for shot dispersion — which are
installed before the action is dispatched into the ordinary input handler.

**Invariants** — the seeds are the mechanism by which the server and every client compute
the *same* bullet spread from the same shot. Without them, a hit on one machine is a miss
on another. This is the only place in the actor where determinism is bought explicitly
rather than assumed, and a rebuild's network protocol must carry the same idea whatever its
wire format.

### equip, eat, activate

**Contract** — one case covering five messages: move an item to a slot, to the belt, to the
rucksack, eat it, or activate an artefact. All five resolve an object, refuse a destroyed
one, refuse a dead player in multiplayer, and then dispatch. Activating an artefact returns
early because an artefact is not required to be an inventory item at that point.

### slot activation, sprint block, weapon hiding

- **activate slot** — makes a numbered slot's item the active one.
- **disable sprint** — a signed *counter*, not a flag. Several systems can each forbid
  sprinting and each must be able to withdraw its own veto; sprint is possible only when
  the counter is at or below zero, and the sprint request is cleared the moment it goes
  positive. A rebuild must keep the counter; a boolean here is a real bug class.
- **weapon hide state** — blocks a set of slots under a named reason, the same tagged
  mechanism the vehicle path uses.

### move, heal

- **move actor** — teleport. See below.
- **max power** — restores stamina to full, clears every wound and clears the blood decals.
  One message, three effects, because a script that grants full stamina means all three.
- **max health** — sets health to the maximum.

### attach and detach holder

**Contract** — the deferred half of the use gesture in
[`ActorInput.cpp`](ActorInput.cpp.md). Attach resolves the holder, asserts the actor is in
none, and refuses a holder already engaged by someone else. Detach verifies the named
holder is the one occupied and leaves it.

**Invariants** — the engaged check is what prevents two players from entering one car. It
is a check on the *holder*, not on the actor, which is the correct place for it.

### head-shot particle, jump

The head-shot case forwards to the particle handler. The jump case is **entirely commented
out**: it once forced a remote actor into a jump from a supplied direction and impulse.
Network jump is now reproduced by the ordinary state replication, and a rebuild should omit
the message.

## `MoveActor`

**Contract** — teleports the actor to a position and a facing. Sets the body yaw, the torso
yaw and pitch, and their unaffected counterparts — the versions before camera effectors —
clears the lean, points the active camera at the new facing, forces the transform through
the physics system, and cancels any network interpolation in progress.

**Invariants** — cancelling interpolation is mandatory. A teleport that leaves interpolation
running makes the actor visibly slide from the old position to the new one over the next
several frames, through whatever geometry lies between.

**Notes** — the camera's yaw is set to the *negative* of the torso yaw and the pitch is set
to the negative of the supplied pitch. Two sign conventions meet here: the message's, the
skeleton's and the camera's. A rebuild should choose one convention and convert at the
message boundary.

The roll is explicitly zeroed rather than carried over, with the carrying version left in
the source as a comment. A teleport always ends with the player upright.
