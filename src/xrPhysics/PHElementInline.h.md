# src/xrPhysics/PHElementInline.h

> The four small frame conversions an element performs constantly: between the body's origin (which is the centre of mass) and the element's visible frame, and between the element's world placement and the bone transform the skeleton expects.

**Needs** — [`PHElement.h`](PHElement.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHValideValues.h`](PHValideValues.h.md)
**Used by** — [`PHElement.cpp`](PHElement.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHShell.cpp`](PHShell.cpp.md) · [`PHShellActivate.cpp`](PHShellActivate.cpp.md)
**Tier floor** — T2: pure transform algebra, nothing device- or format-facing. It is inline in the original only because it runs once per bone per frame.

## Purpose

An element's rigid body is positioned *at its centre of mass* — that is what the dynamics library
demands, because a body whose origin is not its centre of mass needs a coupling term in every
integration step. Everything else in the engine wants the element's *authored* frame: the bone's
origin, not the mass centre. Every read and write of an element's placement therefore passes
through a shift by the local mass centre, and these are the two directions of that shift plus the
two frame conversions that sit beside it.

Getting the direction of the shift backwards is the classic way to make every physical object in
the game render offset from its own collision, and the offset is small enough to look like a bug
somewhere else entirely.

## `InverceLocalForm`

**Contract** — produces the transform that maps the *body's* frame to the *element's* frame: a pure
translation by the negated local mass centre. Overwrites its argument; reads nothing else.

```text
FUNCTION inverse_local_form() -> transform
  RETURN translation_by(-local_mass_center)
```

## `MulB43InverceLocalForm`

**Contract** — right-multiplies a transform by the above without building it. Given a transform that
places the body, yields the transform that places the element.

```text
FUNCTION shift_body_frame_to_element_frame(INOUT m)
  offset := -local_mass_center
  m.origin := m.origin + m.rotation·offset      # rotate the offset into m's frame first
```

**Notes** — the offset is expressed in the element's own frame, so it must be rotated by the
transform before it is added to the origin. A rebuild that adds the offset in world space gets an
object whose render position swings as it tumbles.

## `CalculateBoneTransform`

**Contract** — expresses this element's world placement relative to its shell's root, which is the
form the skeleton wants for a bone.

```text
FUNCTION calculate_bone_transform() -> transform
  RETURN inverse(shell.world_transform) · this_element.world_transform
```

**Notes** — this is the value written into the bone by `bones_callback`, and the reason a physics-
owned bone's transform must not be composed with its parent again: it is already absolute within
the shell.

## `ActivatingPos`

**Contract** — the one-time handover from animation to physics, performed at the *first* bone
evaluation after activation rather than at activation itself. Teleports the body onto the animated
bone, clears the activating flag, and — if this element is the shell's root — records where the
game object's origin sits relative to this bone so the whole object can be placed from the root
body afterwards. Asserts that the resulting placement is a proper rotation inside the world
boundaries.

```text
FUNCTION activating_pos(bone_transform)
  to_bone_pos(bone_transform)              # place the body, reset both interpolation samples
  activating := false
  IF this element has no parent element THEN
    shell.set_object_vs_shell_transform(bone_transform)
  REQUIRE bone_transform is a rotation and its origin is inside the world boundaries
```

**Notes** — the deferral is the whole point. Activation is requested at an arbitrary moment; the
animation system holds the authoritative pose only when the skeleton is evaluated. Placing at
activation would start a ragdoll from a stale pose, and the transition pops visibly. See
[`PHElement.cpp`](PHElement.cpp.md) for the ownership contract this is half of.
