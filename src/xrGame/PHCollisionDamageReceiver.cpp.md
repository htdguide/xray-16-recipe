# src/xrGame/PHCollisionDamageReceiver.cpp

> Turns a physical collision against a particular bone into a damage event, scaled by a per-bone factor authored in the model.

**Needs** — [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrPhysics/IPhysicsShellHolder.h`](../xrPhysics/IPhysicsShellHolder.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/Geometry.h`](../xrPhysics/Geometry.h.md) · [`xrPhysics/icollisiondamagereceiver.h`](../xrPhysics/icollisiondamagereceiver.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHCollisionDamageReceiver.h`](PHCollisionDamageReceiver.h.md)
**Tier floor** — T2: a contact callback invoked from inside the physics step, doing a small lookup and emitting a message

## Purpose

Being hit by a falling crate should hurt, and it should hurt differently depending on where it
lands. The physics world knows a contact happened and how hard; it knows nothing about damage.
This mixin is the bridge: it reads a per-bone damage table out of the model's own authored
data, subscribes the listed bones' collision shapes to a contact callback, and converts a
contact above a threshold into an ordinary strike hit.

Because the table lives in the *model file* rather than in the configuration, the damage
profile travels with the skeleton — which is what makes it authorable per creature without a
configuration section per creature.

## State

```text
RECORD CollisionDamageReceiver
  controlled_bones : list<(bone id, factor)>    # invariant: no bone appears twice
```

**Invariants** — a bone may appear at most once; a duplicate is a hard failure at load, not a
merge. The list is searched linearly, which is right: it holds a handful of entries and is
consulted from inside the physics step.

## `Init`

**Contract** — reads the model's embedded `collision_damage` section, resolves each key from a
bone name to a bone index, records it with its numeric factor, and subscribes that bone's
collision shape to the damage contact callback. Fails hard on an unknown bone name. Does
nothing if the model has no such section.

```text
FUNCTION Init()
  data = the model's embedded configuration, section "collision_damage"
  IF the section is absent THEN RETURN
  FOR EACH (bone name, factor) IN data
    index = bone index for that name          # FAIL WITH wrong bone name
    record (index, factor)                    # FAIL WITH duplicate bone
    shape = the physics shape for that bone
    IF shape exists THEN subscribe it to the damage contact callback
  END FOR
```

**Notes** — a listed bone with no physics shape is silently skipped. The assertion that would
have caught that case is commented out in the original, which means a model can name a bone
that can never receive a collision and nothing will say so.

Only the listed bones are subscribed. The rest of the body collides normally and does no
damage — which is the point: a creature's whole skeleton is in the physics world, but only a
few parts are worth being crushed against.

## `CollisionHit`

**Contract** — called for one contact on a subscribed bone. Looks the bone up, scales the
contact power by its factor, discards anything under the threshold, and emits a strike hit
event against the object. Emits no impulse: the physics has already delivered the momentum.

```text
FUNCTION CollisionHit(source_id, bone_id, power, direction, position)
  entry = the controlled-bone entry for bone_id
  IF absent THEN RETURN                      # this bone does not take collision damage
  power = power * entry.factor
  IF power < HIT_THRESHOLD THEN RETURN       # 5; see below

  hit.kind           = HIT
  hit.target         = this object
  hit.who            = source_id
  hit.weapon         = source_id             # the colliding object is both cause and weapon
  hit.direction      = direction
  hit.power          = power
  hit.bone           = bone_id
  hit.position_in_bone_space = position
  hit.impulse        = 0                     # the physics already applied it
  hit.type           = STRIKE
  send it as an entity event
```

**Invariants** — the impulse is deliberately zero. Delivering one here would double the
momentum, because the contact that caused this callback is also being solved by the physics
world in the same step. This is the mirror of the explosion hit in
[`Mincer.cpp`](Mincer.cpp.md), which carries an impulse and no power.

**Notes** — the threshold is a fixed five, not a tunable. Below it, every footstep, every brush
against a wall and every settling ragdoll bone would generate a hit event, and the event path
is not free. The value is a noise floor rather than a game balance number, which is why it is
in code, but a rebuild should still make it configurable — five is calibrated against the
original's mass and impulse units and will not transfer to a different physics library.

Naming the colliding object as both the responsible party *and* the weapon is what lets the
death message read "killed by a falling crate"; there is no separate notion of an environmental
cause.

## `Clear`

**Contract** — empties the bone table. Does **not** unsubscribe the contact callbacks.

**Notes** — the unsubscribe loop is present but commented out. In practice the shapes are
destroyed alongside the table, so the dangling subscription is never invoked; a rebuild whose
shapes can outlive the receiver must restore it.
