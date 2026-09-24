# src/xrGame/physics_game.cpp

> The bridge from a physics contact to what the player sees and hears: a scuff mark, a spark, a thud — chosen by the two surfaces' material pair and gated by impact speed and camera distance.

**Needs** — [`physics_game.h`](physics_game.h.md) · [`PhysicsGamePars.h`](PhysicsGamePars.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`PHSoundPlayer.h`](PHSoundPlayer.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrPhysics/PHCommander.h`](../xrPhysics/PHCommander.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`Include/xrRender/WallMarkArray.h`](../Include/xrRender/WallMarkArray.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`physics_game.h`](physics_game.h.md)
**Tier floor** — T1: runs inside the physics solver's contact callback, on raw contact records, with no allocation permitted

## Purpose

When two things touch, the player should see a mark, see a puff, and hear a knock — and which
of the three, and which mark and which puff, comes from the *pair* of materials involved, not
from either one alone. This file is where that lookup happens and where the three effects are
scheduled.

It is written under one hard constraint that shapes everything on the page: **it runs inside
the physics step**. Creating a particle emitter, adding a decal or starting a sound from
inside the solver is not allowed, so each effect is packaged as a deferred command and handed
to a queue the level drains after the step. Every class on this page is one such package.

## State

`Stateless` — the deferred command objects carry their own data.

## Constants

```text
particle_effect_distance = 70 real    # metres from the camera; beyond this, no particles
sound_effect_distance    = 70 real    # metres from the camera; beyond this, no sound
mass_limit               = 10000 real # a conventional mass ceiling used only to scale
                                      # collision volume; nothing enforces it
liquid_particle_spacing  = 0.2 real   # metres, horizontal only
liquid_effect_lifetime   = 3000 int   # milliseconds a pending splash stays pending
wallmark_radius          = 0.09 real  # metres
```

Both distance gates are compared as squared distances against squared thresholds, so no square
root is taken per contact. The thresholds are equal, but are separate constants because the
two effects are conceptually independent.

The mass ceiling is explicitly conventional — the source says so — and appears only in the
volume formula below. Nothing rejects a heavier body.

## `TContactShotMark` — the contact callback

**Contract** — called by the physics layer for one contact between a static triangle and a
body. Looks up the material pair, and schedules up to three effects. Returns nothing.
Allocates only deferred command objects. Must not touch the render, audio or particle systems
directly.

It exists in two instantiations differing **only** in their threshold set: one for ordinary
bodies and one for character bodies. Characters generate contacts constantly while walking, so
their thresholds are higher; separating them at the type level rather than branching keeps the
thresholds out of the inner loop.

```text
FUNCTION on_contact(triangle, contact)
  IF the physics layer cannot supply effect parameters for this contact THEN RETURN
    # yields: the body's user data, an impact speed measure, and whether the
    # contact normal points the wrong way for this body

  camera_distance_sq := squared distance from the contact point to the camera
  pair := the material pair for (triangle's material, body's material)
  IF there is no such pair THEN RETURN          # nothing authored for this combination

  # 1. wallmark — not distance-gated
  IF impact_speed > wallmark_threshold AND the pair has collision marks
    schedule: add a static wallmark at the contact point, radius 0.09,
              on this triangle, against the level's static vertices

  # 2. sound
  IF camera_distance_sq < sound_distance_sq
    static_material := the triangle's material
    IF the material is NOT passable
      IF impact_speed > sound_threshold AND the pair has collision sounds
        volume := min_volume + impact_speed * (max_volume - min_volume)
                  / (sqrt(mass_limit) * linear_speed_limit - sound_threshold)
        play a randomly chosen sound from the pair, positioned at the contact,
          without feedback
    ELSE
      IF the body has a game object AND the pair has collision sounds
        let that object's own physics sound player handle it

  # 3. particles
  IF camera_distance_sq < particle_distance_sq AND the pair has collision particles
    name := a randomly chosen particle effect from the pair
    play_particles(impact_speed, body, contact, invert_normal, static_material, name)
```

**Invariants** — the wallmark is **not** distance-gated while the sound and particles are.
That is deliberate: a mark persists, so a mark made at fifty metres must exist when the player
walks over to look at it, whereas a sound or a puff at fifty metres is gone before anyone
could arrive. A rebuild that gates all three uniformly will produce clean walls in places the
player shot from a distance.

A *passable* material — foliage, a curtain, shallow water — takes a completely different sound
path: rather than the material pair's generic collision sound, the **object** is asked to make
its own noise through its physics sound player. Passable materials are things you walk
through, and what they sound like depends on what walked through them. This is the only branch
on the page that routes to a per-object sound rather than a per-material-pair one.

The sound is played *without feedback*, meaning the caller keeps no handle and cannot stop or
track it. Contact sounds are fire-and-forget by nature, and keeping handles for thousands of
contacts would exhaust the mixer's source budget.

**Notes** — the volume formula linearly maps impact speed from the sound threshold up to a
notional maximum onto the authored volume range. That maximum — the square root of the mass
ceiling times the physics layer's linear speed limit — is dimensionally a momentum-like
quantity, not a speed, so the mapping is not physically principled; it is a curve whose only
justification is that it sounds right across the range of impacts the shipped content
produces. A rebuild may substitute any monotonic map from impact speed to volume and retune.

The formula does not clamp. An impact above the notional maximum yields a volume above the
authored maximum, which the audio device will clip.

Both the sound and the particle effect are chosen by uniform random draw from the material
pair's list, so repeated identical impacts vary. The two draws use slightly different index
conventions in the source — one draws in a half-open range from zero, the other draws over the
count — with the same effect.

## `play_particles` and the liquid case

**Contract** — decides whether to schedule a particle effect for this contact, and by which of
two paths. Splits on whether the *static* material is flagged as liquid.

```text
FUNCTION play_particles(impact_speed, body, contact, invert_normal, static_material, name)
  IF the static material is not liquid
    IF impact_speed > particle_threshold
      schedule an unconditional one-shot particle play
    RETURN

  # liquid: a lower threshold for non-characters, and de-duplication in the plane
  eligible := impact_speed > particle_threshold
           OR (the body is not a character AND impact_speed > particle_threshold / 4)
  IF NOT eligible THEN RETURN

  IF a pending liquid splash already exists within 0.2 metres horizontally
    RETURN                              # one splash per patch of surface
  schedule a de-duplicating liquid particle play
```

**Invariants** — liquid gets a threshold four times lower for non-character bodies, because a
dropped object entering water should splash at a speed at which it would not scuff a wall.
Characters are excluded from the concession: a walking creature produces many slow contacts
with a water surface, and at the reduced threshold every footstep would splash.

The de-duplication distance is measured **horizontally only** — the vertical component of the
separation is ignored. A water surface is a horizontal plane, so two contacts at the same
place and different depths are the same splash. A rebuild using full 3D distance will produce
a column of splashes for an object sinking through the surface.

De-duplication is a *query against the pending command queue*, not against effects already
played: the queue is searched for an unplayed liquid splash near this position, and if one is
found this contact is dropped. So the mechanism only merges splashes generated within the same
drain cycle, which is what the solver produces for one object hitting water.

**Notes** — the queue query works through a small visitor protocol: a comparer is handed to
the queue and each pending entry is asked to compare itself against it, so that the queue does
not need to know the types of the commands it holds. A rebuild with a typed queue does not
need the protocol.

## The deferred commands

**Contract** — three command types, each capturing what it needs at contact time and doing its
work when the queue is drained after the physics step.

- **Particle play** — captures the contact position and normal (negating the normal when the
  body is on the far side of the contact) and the effect name. On run, creates the particle
  object, builds an orthonormal frame whose *up* axis is the contact normal, positions it at
  the contact point, and hands it to the persistent game state's play queue with zero
  velocity.
- **Liquid particle play** — the same, plus two additions: it runs **at most once** even if
  drained repeatedly, and it declares itself obsolete three seconds after creation so it is
  discarded if never drained. It also exposes its position, which is what the de-duplication
  query reads.
- **Wallmark** — captures the position, the triangle and a generated decal description. On
  run, adds a static wallmark of the fixed radius against the level's static geometry.

**Invariants** — the particle frame is built *from the contact normal outward*: the normal
becomes the effect's up axis and the other two axes are generated to be orthonormal to it,
with no preferred roll. A spark or splash is rotationally symmetric about the surface normal,
so an arbitrary roll is correct and free.

The normal negation exists because a contact has one normal and two sides; the effect must
emit away from the surface, and which direction that is depends on which of the two bodies is
being handled. The physics layer reports which, and the flag is applied once at capture time
rather than at run time.

The ordinary particle command is never obsolete and has no run-once guard; only the liquid one
needs both, because only liquid commands linger in the queue waiting to be de-duplicated
against.

## The two installed callbacks

**Contract** — the file's only exports: the ordinary-body callback and the character-body
callback, each the same function with its own threshold set. Both are installed as references
the physics layer holds.

**Invariants** — the two threshold sets are the entire difference. Their values live in the
physics-parameters header and are not this file's content, but the *shape* is: three
thresholds per set, for wallmarks, sounds and particles independently, so that a light scuff
can mark without sounding and a heavy one can do both.
