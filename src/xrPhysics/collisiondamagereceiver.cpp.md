# src/xrPhysics/collisiondamagereceiver.cpp

> Decides how much a collision hurts: the energy of the impact, split between the two surfaces
> by what each material is willing to contribute.

**Needs** — [`icollisiondamagereceiver.h`](icollisiondamagereceiver.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [`PHObject.h`](PHObject.h.md) · [`Physics.h`](Physics.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`icollisiondamagereceiver.h`](icollisiondamagereceiver.h.md)
**Tier floor** — T1: reads the contact record and both bodies' state by layout, inside the
solver's callback.

## Purpose

Two contact callbacks, both answering the same question in different ways: *given that these
two things just hit each other, how much damage does the one I am watching take?*

The interesting decision is not the energy formula — that is shared and lives in
[`Physics.cpp`](Physics.cpp.md) — but the **material apportionment**: how the available damage
is divided between the struck surface and the striking one. That is what makes running into a
concrete pillar hurt and running into a bush not, with the same closing speed.

## Stateless.

## `DamageReceiverCollisionCallback`

**Contract** — installed on the shapes of an entity that can take collision damage. For each
contact, compute the inelastic collision energy, scale it by the striker's material, split it
against the pair of materials, and hand the result to the receiver. Does not reject the
contact — the collision still happens normally.

```text
FUNCTION damage_receiver_contact(inout accept, this_side_is_first, contact, mat_a, mat_b)
  IF either material is marked PASSABLE THEN RETURN      # you cannot be hurt by grass

  self     := the side I am watching        (chosen by this_side_is_first)
  damager  := the other side
  mat_self, mat_damager  likewise

  receiver := self.owner.collision_damage_receiver()
  FAIL WITH "wrong callback" IF receiver is none         # installed on the wrong shape

  damager_factor := mat_damager.bounce_damage_factor
  IF the damager is a CHARACTER
    damager_factor := damager.owner.override_bounce_damage_factor(damager_factor)

  total := mat_self.bounce_damage_factor + damager_factor
  IF total is zero THEN RETURN                           # neither surface deals damage

  power := inelastic_collision_energy(body_of(shape_a), body_of(shape_b), contact.normal)
           * damager_factor / total

  receiver.collision_hit(
      source_id := damager.owner.id (or the "no entity" marker),
      bone_id   := self.bone_id,
      power     := power,
      direction := contact.normal,
      position  := contact.position - origin_of(self.shape))
```

**Invariants** — the split `damager_factor / (self_factor + damager_factor)` is a *fraction*,
so the two surfaces' factors are relative weights, not absolute scales. Doubling both changes
nothing. That is the property that makes the shipped material table tunable one entry at a
time without rebalancing everything.

**Notes** — three decisions worth naming.

*The passable flag short-circuits before any work.* Passable materials — foliage, cloth, the
markers the AI uses — must never produce damage even when the closing speed is enormous, and
checking either side rather than just the striker means a passable *thing* cannot be hurt
either.

*A character damager may override its own material factor.* The player and creatures are not
made of their collision material: a stalker running into you should hurt by what the *game*
says a stalker weighs, not by what the capsule's surface is made of. The callback asks the
owner for the substitution and takes whatever comes back.

*The energy formula is shared with every other impact consumer.* That is deliberate: the same
closing event must produce one energy number whether it is being turned into damage, into a
sound, or into a decal, or the three disagree visibly.

## `BreakableObjectCollisionCallback`

**Contract** — installed on the shapes of a breakable prop. Reports the energy of the striking
*body alone* against the contact plane, as an anonymous hit with no source and no bone.

```text
FUNCTION breakable_contact(accept, this_side_is_first, contact, mat_a, mat_b)
  IF this_side_is_first
    receiver := owner_of(shape_a).collision_damage_receiver()
    body     := body_of(shape_b);  normal_sign := -1
  ELSE
    receiver := owner_of(shape_b).collision_damage_receiver()
    body     := body_of(shape_a);  normal_sign := +1

  power := single_body_energy_along(body, contact.normal, normal_sign)
  receiver.collision_hit(
      source_id := no entity,
      bone_id   := no bone,
      power     := power,
      direction := -contact.normal * normal_sign,   # points INTO the breakable
      position  := contact.position)                # world, unlike the other callback
```

**Invariants** — the sign is chosen so the reported direction points into the breakable
surface from the striker. The contact normal's own orientation depends only on which shape the
collider happened to test first, which is an artefact of the query and not a fact about the
world; the sign flip is what converts it into a fact.

**Notes** — this callback differs from the first in three ways that all follow from what a
breakable *is*. It ignores materials entirely (a window breaks by energy, not by what hit it);
it uses the **single-body** energy rather than the two-body inelastic energy (the breakable is
treated as immovable, so only the striker's kinetic energy matters); and it reports position
in world space rather than relative to the shape. The last is an inconsistency between two
implementations of one interface and a rebuild should settle on one convention — world — for
both.

Notably it also does not consult a damage *threshold*. The original once compared the energy
against a per-object threshold here and now leaves that decision entirely to the receiver.
That is the right layering: physics reports energy, the game decides what breaks.
