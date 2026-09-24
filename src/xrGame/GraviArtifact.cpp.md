# src/xrGame/GraviArtifact.cpp

> The gravitational artefact: while lying loose in the world it repeatedly kicks itself upward so that it hovers unsteadily just above the ground.

**Needs** — [`GraviArtifact.h`](GraviArtifact.h.md) · [`Artefact.h`](Artefact.h.md) · [`Level.h`](Level.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame impulse application against a rigid body and a ray query; no layout or device concern

## Purpose

Most artefacts are inert props with a carry effect. This one has a visible idle
behaviour: dropped on the ground it bobs, never settling, which is the whole point of the
item's identity. The behaviour is one rule — *if there is ground within a configured
distance below me, push me up* — and it is deliberately a physics impulse rather than a
scripted animation, so the artefact still collides, rolls and is knocked around normally.

The file also carries the attachment rule for the one multiplayer mode where an artefact
is carried visibly on a player's back.

## State

```text
RECORD GraviArtefactState
  jump_height  : real     # ray length probed downward; 0 disables hovering entirely
  energy       : real     # loaded but never read — see Notes
```

Scheduling: the artefact asks the scheduler for a comparatively tight update interval
(roughly 20–50 milliseconds between updates rather than the default coarse rate),
because the hover is visible and a slow update rate makes it stutter.

## `Load`

**Contract** — reads the section after the base artefact has read it. The hover distance
is optional: when the section omits it the field stays zero and the hover rule is
disabled, which is how every non-hovering artefact that reuses this class behaves.

## `UpdateCLChild`

**Contract** — the per-frame client-side hook, called while the physics world is *not*
stepping (applying an impulse mid-step would corrupt the solve). Two mutually exclusive
cases: the artefact is loose in the world and visible, or it is attached to a carrier.

**Invariants** — the impulse is scaled by the artefact's own mass and by the frame's
elapsed time, so the resulting velocity change is frame-rate independent and
mass-independent; a heavier artefact hovers identically to a lighter one.

```text
FUNCTION update_client_child()
  REQUIRE physics world is not mid-step

  IF visible AND has a physics body THEN
    IF jump_height > 0 THEN
      hit = raycast(from: position, direction: straight down,
                    max_distance: jump_height, hits: static and dynamic)
      IF hit THEN
        # ground is close enough: kick upward
        apply_impulse(direction: straight up,
                      magnitude: 30 * frame_seconds * mass)
      # no hit means free fall; gravity brings it back into range and it bobs
    RETURN

  ELSE IF attached to a carrier THEN
    transform = carrier.transform
    IF game mode is artefact-hunt AND carry_bone is set THEN
      carrier.skeleton.evaluate_pose()
      bone = carrier.skeleton.transform_of(carry_bone)
      bone.translate_by(-0.1, 0, -0.3)      # sit the artefact in the rucksack, not on the bone's origin
      transform = bone composed with carrier.transform
```

**Notes**

- The upward push is applied every update while ground is in range, so the artefact
  never reaches equilibrium: it overshoots, leaves the ray's range, falls back in and is
  pushed again. The *instability* is the effect, not a bug to damp out.
- The impulse coefficient (30) and the rucksack offset are tuning constants with no
  derivation in the source; they are what looked right.
- The carry-bone branch is gated on one multiplayer mode because only that mode shows a
  carried artefact on the player's body. In every other mode a carried artefact is
  invisible inventory contents.
- The "energy" parameter is initialised to one and never used; a rebuild should drop it.
