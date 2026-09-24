# src/xrGame/aimers_weapon.cpp

> Aims a held weapon rather than a bone: works out where the muzzle actually is, from the weapon's configured mounting and fire point, then rotates two of the carrier's bones so the bullet's path passes through the target.

**Needs** — [`aimers_weapon.h`](aimers_weapon.h.md) · [`aimers_weapon_inline.h`](aimers_weapon_inline.h.md) · [`aimers_base.h`](aimers_base.h.md) · [`Weapon.h`](Weapon.h.md) · [`GameObject.h`](GameObject.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`aimers_weapon.h`](aimers_weapon.h.md)
**Tier floor** — T2: geometry plus a configuration read on a per-decision path.

## Purpose

A weapon is not part of the character's skeleton. It hangs off a pair of hand bones, at an
offset the weapon's own configuration section states, and the bullet leaves from a fire
point stated in the same section. So "where does this gun point" cannot be read from any
one bone; it has to be reconstructed. This file reconstructs it and then feeds the result
to the shared aiming solver.

## State

```text
RECORD WeaponAimer               # extends the aimer base
  weapon         : reference to the weapon being aimed
  bone_ids       : five skeleton bones:
                     link 0, link 1        # the two carrier bones that will be rotated
                     weapon anchor 0,      # the pair of hand bones whose line defines
                     weapon anchor 1       #   the weapon's mounting axis
                     parent of anchor 0    # sampled so the anchor's frame is complete
  bones          : their world poses
  local_bones    : their poses in the skeleton's own space
  result         : two corrections, one per carrier bone

# Invariant: the parent of the first weapon anchor must exist. A weapon anchor with
#   no parent means the model is not a carrier model, which is a data error.
```

## Construction

**Contract** — Resolves the five bones, solves the first carrier link, splits its rotation
in half, installs that half on the skeleton, solves the second link against the corrected
pose, and restores the skeleton. Reads the weapon's configuration section three times. The
whole answer exists once construction returns.

```text
FUNCTION construct(...)
  resolve the four named bones; the fifth is anchor 0's parent
  compute_bone(link 0)
  halve link 0's Euler angles                # each of the two links takes half
  install it as link 0's skeleton hook
  compute_bone(link 1)                       # solves the residual
  restore link 0's previous hook
```

**Invariants** — The same distribute-and-recurse shape as the chain aimer, fixed at two
links and written out rather than recursive. Halving by Euler angles is the same
approximation, and the same argument applies: link 1 re-solves the residual exactly, so
the split's inexactness never accumulates.

## `compute_bone`

**Contract** — Reconstructs the weapon's world-space fire ray under the candidate
animation, and asks the shared solver for the rotation of one carrier bone that puts the
target on that ray. The result is converted into the object's own frame before it is
stored, so the skeleton can apply it directly.

```text
FUNCTION compute_bone(link)
  sample link, both weapon anchors and anchor 0's parent under the candidate motion

  # 1. The weapon's mounting frame, built from the two anchor bones: the line
  #    between them is the weapon's long axis, and anchor 0's own up axis
  #    resolves the roll about it.
  axis  = normalize(anchor1.position - anchor0.position)   # forward; substituted with
                                                           #   the object's forward if
                                                           #   the anchors coincide
  right = cross(anchor0.up_axis, axis)
  up    = normalize(cross(axis, right))
  mount = frame (right, up, axis) placed at anchor0.position, taken into world space

  # 2. The weapon's own offset from that mounting, and its fire point, are
  #    configuration, not geometry.
  offset  = rotation from the weapon section's "orientation" (degrees)
            translated by the weapon section's "position"
  muzzle_frame = mount composed with offset
  ray_origin    = muzzle_frame applied to the section's "fire_point"
  ray_direction = muzzle_frame's forward axis

  # 3. Solve, then express the answer in object space.
  result[link] = aim_at_position(pivot = link.position, ray_origin, ray_direction)
  result[link] = inverse(reference) . result[link] . reference
```

**Invariants** — The mounting frame must be orthonormalized in that order — forward from
the anchors, right from the cross with the anchor's up, up from the cross of the first two
— because the two anchor bones fix only an axis and a rough roll; taking the anchor's up
axis directly would leave the frame non-orthogonal whenever the hands are not exactly as
the model's rest pose has them.

**Notes** — The three configuration values are read from the weapon's own section on every
solve: the mounting position, the mounting orientation in degrees, and the fire point. A
note in the source says the weapon should be asked for this rather than the configuration
re-parsed, and it is right — this is a per-decision path and the section lookup is a string
hash and a parse each time. A rebuild should cache the muzzle offset on the weapon at load.

That the aiming reference is the *configured* fire point rather than a bone on the weapon's
own model is the important decision: the weapon model's muzzle bone and the ballistic
origin are separate things, and the ballistic one is what must be aimed, so that where a
character is seen pointing and where his bullets go agree.
