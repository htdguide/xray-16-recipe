# src/xrPhysics/PHSimpleCharacterInline.h

> How hard a collision hurt — two different models for hitting the level and being hit by an object — and the rule that decides whether you are standing *on* a surface or *in* it.

**Needs** — [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md) · [`ElevatorState.h`](ElevatorState.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`PHSimpleCharacter.h`](PHSimpleCharacter.h.md)
**Tier floor** — T2: energy and velocity arithmetic over a contact, plus a material-flag decision.

## Purpose

Three procedures that are inline only because they run once per contact per character per step. They
are substantive: the damage model is nowhere else, and it is the thing that decides whether a fall
kills you.

The two damage models differ because the situations are genuinely different. Hitting the *level* is
a one-sided event — the world does not move, all the energy is yours, and the surface decides how
much of your speed becomes damage. Being hit by an *object* is a two-body collision, and what matters
is how much kinetic energy the collision cannot conserve.

## `UpdateStaticDamage`

**Contract** — given a contact with the static world, compute an *effective velocity* and store it if
it exceeds what has already been recorded this step. The stored record is the worst of the step.

```text
FUNCTION update_static_damage(contact, surface_material, this_side_is_first)
  v      := the character's velocity
  normal := the component of v along the contact normal, as a magnitude
  plane  := the component of v in the contact plane

  IF the surface is PASSABLE THEN
    effective := |v| · surface.bounce_damage_factor       # the WHOLE speed counts
  ELSE
    effective := MAX(plane · surface.friction, normal) · surface.bounce_damage_factor

  IF effective > the recorded contact velocity THEN record this contact as the worst
```

**Notes** — the choice of what counts is the decision, and it is a good one.

*Against a solid surface, damage comes from whichever is worse: hitting it head-on, or scraping along
it hard.* The head-on part is the normal component — an ordinary fall. The scraping part is the
in-plane component **scaled by the surface's friction**, so sliding along ice at speed hurts far less
than sliding along concrete at the same speed. That is what makes a glancing fall down a slope
survivable and a flat landing not.

*Against a passable surface, the whole speed counts.* Grass, water and foliage reject their contacts,
so the character continues through them; the damage is the full impact speed scaled by the material's
own damage factor — which is small for water and near zero for grass. This is how landing in water
from a height still hurts, but less.

The per-material `bounce_damage_factor` is authored data: it is the entire tuning surface for fall
damage, one number per material, and a rebuild must read it from the shipped material table. See
[`xrMaterialSystem`](../xrMaterialSystem/README.md).

The recorded *sign* says which way the contact normal points relative to this character, so the
damage direction can be reconstructed later from the stored contact alone.

## `UpdateDynamicDamage`

**Contract** — given a contact with another body, compute how much kinetic energy the collision
destroys, convert the character's share of it back into an effective velocity, and record it if it
is the worst of the step. Objects marked as being destroyed are ignored.

```text
FUNCTION update_dynamic_damage(contact, other_material, other_body, this_side_is_first)
  n := the contact normal
  my_normal_speed    := this character's velocity along n
  their_normal_speed := the other body's velocity along n

  # separating, not approaching: no collision energy
  IF the two are moving apart along the normal THEN RETURN

  my_energy    := ½ · my_mass    · my_normal_speed²
  their_energy := ½ · their_mass · their_normal_speed²

  # the kinetic energy that a perfectly inelastic collision would KEEP:
  # the energy of the combined momentum moving as one mass
  combined := (my_mass·my_normal_speed + their_mass·their_normal_speed)² / (2·(my_mass+their_mass))

  accepted := my_energy·my_damage_factor + their_energy·object_damage_factor − combined
  IF accepted > 0 THEN
    effective := sqrt(2·accepted / my_mass) · other_material.bounce_damage_factor
  ELSE
    effective := 0

  IF effective > the recorded contact velocity AND the other object is not being destroyed THEN
    record this contact, the other object's identity and its own damage hook
```

**Invariants** — the model is the classic one: *the energy a collision destroys is the total kinetic
energy minus what the combined momentum can carry away.* That difference is the deformation energy,
and it is the only physically meaningful measure of how hard a collision was. Two bodies moving at
the same speed collide with zero energy regardless of how fast they are going, which is exactly
right — a character in a moving lift is not being crushed by it.

**Notes** — the two damage factors are weights on each side's contribution, and they exist so the game
can make a character's own momentum count for more or less than the object's. The character's factor
is per-character (a heavy creature can be made harder to hurt); the object factor is a global
constant. The weighting is applied to the energies *before* the momentum term is subtracted, which
means the result is not an energy any more — it is a tuned quantity with the units of one. That is
acceptable because the outcome is immediately converted back into a velocity and scaled by an
authored per-material factor anyway; the whole chain is calibrated end to end against the shipped
data, and a rebuild must reproduce it rather than re-derive it.

The separating test is the guard that stops a contact being counted twice: a collision generates
contacts for several steps as it resolves, and only the approaching ones are real impacts.

Recording the other object's own **hit callback** alongside its identity is what lets the damage be
routed through that object's rules — a vehicle applies vehicle damage, a rock applies blunt damage —
without this file knowing anything about them.

## `foot_material_update`

**Contract** — decides which material the character reports as being underfoot, given the material of
the contact and the material recorded on the foot shape.

```text
FUNCTION foot_material_update(contact_material, foot_material)
  IF the ladder state claims the material THEN RETURN       # a ladder overrides the ground
  IF the currently reported material is PASSABLE and no re-check was requested THEN RETURN

  clear the re-check request
  IF the contact's material is PASSABLE THEN
    IF it is also INJURIOUS THEN record it as the injurious material
    ELSE                        report it as the material underfoot
  ELSE
    report the FOOT SHAPE's material                        # the solid surface, not the contact
```

**Invariants** — a passable material, once reported, **sticks** for the rest of the step unless a
re-check was explicitly requested. That is what stops a character wading through water from
alternating between "water" and "riverbed" as contacts arrive in arbitrary order — the water wins,
because you are *in* it.

**Notes** — the distinction the whole procedure encodes: **passable means you are standing in it,
solid means you are standing on it.** Water, grass and foliage are passable; they are what the
footstep sound should be, and they are what an injurious surface is. Anything else reports the
surface the foot shape is actually resting on.

An *injurious* passable material is recorded separately rather than as the material underfoot,
because the two are different questions: what you hear when you step, and what is burning you. A
character can be standing on a riverbed, in acid.

The re-check request is set once per step by the controller's post-solve pass, so the first contact
of each step always gets to set the material and later ones only override it under the rules above.
