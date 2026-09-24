# src/xrPhysics/PhysicsExternalCommon.cpp

> Extracts the impact strength and the struck surface from a contact, for whoever wants to draw a decal or play a sound.

**Needs** — [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`MathUtilsOde.h`](MathUtilsOde.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PhysicsExternalCommon.h`](PhysicsExternalCommon.h.md)
**Tier floor** — T1: reads a contact record and a body's mass by layout.

## Purpose

Stateless. One function, used by the game layer's contact-mark handlers: given a raw contact, work
out *which side is the moving one*, how hard the impact was, and whether the contact normal needs
inverting to point the right way for a decal.

## `contact_shot_mark_effect_params`

**Contract** — reads a contact and reports the moving side's user data, an impact magnitude, and
whether the contact normal points away from that side. Returns false when neither side has a body,
which means there is nothing to draw an impact for.

```text
FUNCTION contact_effect_params(contact) -> optional<(user_data, magnitude, invert_normal)>
  body := body_of(contact.shape_a)
  IF body EXISTS THEN
    user_data := user_data_of(contact.shape_a) ; invert_normal := false
  ELSE
    body := body_of(contact.shape_b)
    user_data := user_data_of(contact.shape_b) ; invert_normal := true
  IF body DOES NOT EXIST THEN RETURN none

  point_velocity := velocity of body at contact.position
  magnitude := |dot(point_velocity, contact.normal)| · sqrt(body.mass)
  RETURN (user_data, magnitude, invert_normal)
```

**Notes** — the magnitude is normal velocity times the **square root** of mass, not mass. That is
not momentum and not energy; it is chosen so the visual and audible scale of an impact grows
sub-linearly with mass, because a linear scale makes heavy objects produce absurd effects and an
energy scale (velocity squared) makes fast light objects do the same. It is a presentation curve,
not physics, and a rebuild is free to choose its own — but should choose one deliberately.

The normal inversion exists because the collider's contact normal has an orientation determined by
which shape was passed first, which is an artefact of the query and not a fact about the world. The
caller needs a normal pointing out of the *struck* surface toward the striker.
