# src/xrGame/BastArtifact.cpp

> The artefact that fights back: shoot it, and it charges up and hurls itself repeatedly at whoever is standing near.

**Needs** — [`BastArtifact.h`](BastArtifact.h.md) · [`Artefact.h`](Artefact.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a physics contact callback and per-frame impulses

## Purpose

An artefact with an offensive behaviour driven entirely by physics. It is worth reading for
the *shape* rather than the specifics, because the same shape recurs across the chapter's
reactive objects: **a stored energy pool that damage fills and time drains, a touch sense
that maintains a list of nearby creatures, a contact callback that reacts to collisions, and
a per-frame impulse that steers the object toward a chosen victim.**

## State

```text
RECORD BastArtefact                      # extends artefact, adds a touch sense
  impulse_threshold  : real   # a hit below this does not wake it
  radius             : real   # the touch sense's radius: who counts as "nearby"
  strike_impulse     : real   # the impulse spent per strike, and the energy per strike
  energy             : real   # the charge; invariant: 0 <= energy <= energy_max
  energy_max         : real
  energy_decay       : real   # per second
  particle_name      : text
  striking           : bool   # armed and looking for a victim
  alive_nearby       : list<creature>     # maintained by the touch sense
  attacking          : optional<creature> # the chosen victim; none while unarmed
  last_hit           : optional<creature> # who it last struck, to avoid repeating
```

**Invariants** — the artefact is *useful* — that is, pickable up — only while its energy is
zero. A charged artefact cannot be picked up, which is the safety rule that makes the whole
behaviour playable.

The attacking and last-hit references are cleared at spawn and at destroy, and the touch
sense maintains the nearby list. Note that unlike
[`BlackGraviArtifact.cpp`](BlackGraviArtifact.cpp.md), this class does **not** implement the
reference-release hook, so a creature destroyed while in the nearby list leaves a dangling
entry. That is a real defect; a rebuild with safe references does not inherit it.

## `Load`

**Contract** — reads all six tuning values. All are required; the constructor's defaults are
never used by shipped data.

## `Hit`

**Contract** — the trigger. A hit whose *impulse* exceeds the threshold, with at least one
creature nearby, arms the artefact, clears the current victim, and adds energy proportional
to the strike impulse times the incoming impulse, capped at the maximum. The hit is then
passed on to the base class **with its impulse zeroed**.

**Invariants** — zeroing the impulse is the decision: a bullet that wakes the artefact must
not also knock it flying, because the artefact's own steering is what should move it. The
damage is still applied; only the physical push is suppressed.

**Notes** — the threshold is on impulse, not damage, so a heavy slow impact wakes it and a
fast small one may not.

## `UpdateCLChild`

**Contract** — the per-frame behaviour, running only while the artefact is loose and has a
body. Four independent steps.

```text
FUNCTION update()
  energy -= decay * frame_seconds, floored at 0

  IF loose AND has a body THEN
    # 1. Choose a victim, once, when armed.
    IF armed AND no victim AND somebody is nearby THEN
      IF more than one candidate THEN
        pick a random one, REJECTING the last one struck, until a different one is found
      ELSE pick the only one
      victim = that

    # 2. Steer toward the victim while there is charge to spend.
    IF a victim exists THEN
      IF the victim is alive AND energy > strike_impulse THEN
        energy -= strike_impulse
        direction = victim.centre - own position, with a small random upward jitter
        apply an impulse along it, scaled by frame time and by own mass
      ELSE
        drop the victim and disarm      # out of charge, or the victim died

    # 3. Emit a particle burst with a probability proportional to the charge.
    IF energy > 0 AND random(0,1) < energy / (strike_impulse * 100) THEN
      spawn a one-shot particle effect at the artefact's transform

  ELSE IF carried THEN follow the carrier's transform exactly
```

**Invariants** — the impulse is scaled by the artefact's own mass, so the resulting
acceleration is mass-independent; the artefact flies the same way regardless of how heavy
its section makes it.

The victim-choosing loop rejects the previously struck creature, which is what makes the
artefact bounce between two people rather than pinning one. With exactly one candidate that
rejection is skipped — otherwise the loop would never terminate.

**Notes**

- The particle probability's divisor is `strike_impulse * 100`, which is also the
  constructor's default for the maximum energy — so the burst rate is "fraction of a full
  charge" only when the data agrees with that default. A rebuild should divide by the
  configured maximum.
- Energy is spent per *frame* while steering, not per strike, so the artefact's flight time
  is frame-rate dependent in a way the impulse is not. That is a genuine inconsistency and
  the impulse's own frame-time scaling does not compensate for it.

## `BastCollision`

**Contract** — called from the contact callback when the artefact, while attacking, touches
something. If the thing is a living creature: clear the victim, record it as the last
struck, and re-arm so the next update chooses a new target.

**Notes** — a branch computes whether to re-arm based on how many creatures are nearby and
is then overwritten by an unconditional re-arm on the next line. The computed branch is
dead; reproduce the unconditional behaviour.

Two lines that would have zeroed the artefact's velocity on impact are commented out. With
them the artefact would stop dead on each hit; without them it ricochets.

## `ObjectContactCallback`

**Contract** — the physics contact hook, installed on the artefact's body. Receives a
contact between two geometries and must identify which side is the artefact and which is a
creature.

```text
FUNCTION on_contact(contact)
  resolve both geometries' owning objects; IF either is missing THEN RETURN
  artefact = whichever side is a bast artefact; IF neither THEN RETURN
  IF that artefact is not attacking THEN RETURN
  creature = whichever side is a living creature (may be none)
  artefact.on_collision(creature)
```

**Invariants** — the callback is static and receives no context, so it must recover the
artefact from the contact itself. Both orderings must be handled, because the physics layer
does not guarantee which geometry is first.

**Notes** — the "is it attacking" gate is checked here rather than in the handler so that a
dormant artefact costs nothing per contact, and a dormant artefact collides with the world
constantly.

## `setup_physic_shell`

**Contract** — after the base class builds the body, registers the artefact as the body's
owning object, installs the contact callback, and **clears** the generic contact callback.
The two are alternatives and installing both would deliver every contact twice.

## the touch sense

**Contract** — membership is "a living creature". Entering adds to the nearby list, leaving
removes. The sense is refreshed each scheduled update at the artefact's configured radius.

**Notes** — the refresh happens on the *scheduled* path while the behaviour runs on the
per-frame path, so the nearby list can be up to a scheduling interval stale. For a
ten-metre radius that is acceptable.

## `Useful`

**Contract** — false while charged. See the invariant above.
