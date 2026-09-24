# src/xrGame/BlackGraviArtifact.cpp

> A hovering artefact that, when struck hard enough, detonates a gravitational shockwave that throws and injures everything around it.

**Needs** — [`BlackGraviArtifact.h`](BlackGraviArtifact.h.md) · [`GraviArtifact.h`](GraviArtifact.h.md) · [`Explosive.h`](Explosive.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a radial query with line-of-sight tests, then hit events

## Purpose

The hovering artefact from [`GraviArtifact.cpp`](GraviArtifact.cpp.md) with one behaviour
added: a hit above an impulse threshold arms it, and the next frame it releases a radial
shockwave. It is the chapter's clearest example of a **line-of-sight-attenuated radial
effect**, which is the same machinery an explosion uses, and the reason to read it is that
machinery rather than the artefact.

## State

```text
RECORD BlackGraviArtefact                # extends the hovering artefact, adds a touch sense
  impulse_threshold : real   # a hit below this does nothing
  radius            : real   # the shockwave's reach and the sense's radius
  strike_impulse    : real   # the impulse at the centre
  armed             : bool   # set by a hit, cleared by the strike
  particle_name     : text
  nearby            : list<physical object>   # maintained by the touch sense
  ray_results       : reusable query buffer
```

**Invariant** — the nearby list excludes artefacts, so one of these cannot set off another
and a cluster of artefacts does not chain-react.

## `net_Spawn`

**Contract** — spawns the base artefact and then creates a permanent decorative particle
effect at the artefact's position, scaled down to seven tenths. The effect is created
detached and never referenced again.

**Notes** — the effect's name is a fixed literal rather than configuration, and the scale is
a bare constant. A rebuild should make both section keys, which costs nothing and removes
the only hard-coded asset reference in the class.

## `Load`

**Contract** — reads the four tuning values on top of the base artefact's. All required.

## `Hit`

**Contract** — a hit whose *impulse* exceeds the threshold arms the artefact, and the hit is
passed on with its impulse zeroed so the shot does not also knock the artefact away. Exactly
the trigger pattern of [`BastArtifact.cpp`](BastArtifact.cpp.md).

## `UpdateCLChild`

**Contract** — runs the base hover behaviour, then, if armed and loose and physical:
refreshes the touch sense at the strike radius, releases the shockwave, spawns a one-shot
particle effect, and disarms. A carried artefact instead follows its carrier's transform.

**Invariants** — the sense is refreshed *immediately before* the strike rather than relying
on the periodic refresh, because the strike's victim set must be current at the moment of
detonation.

## `GraviStrike`

**Contract** — the shockwave. For every nearby physical object, computes a falloff impulse,
attenuates it by line of sight, decides whether the object also takes damage, and sends a
hit event per affected bone.

```text
FUNCTION strike()
  clear the ray-result buffer            # reused across the whole sweep, not per object

  FOR EACH object IN nearby
    centre    = the object's visual centre, or its origin if it has no visual
    direction = centre - own position; distance = |direction|

    # Quadratic falloff, reaching zero exactly at the radius and going
    # NEGATIVE beyond it — so an object outside the radius is skipped by the
    # positive test below rather than by a distance test.
    impulse = 100 * strike_impulse * (1 - (distance/radius)^2)

    IF impulse > small THEN
      impulse *= line_of_sight_fraction(from: own position, to: object, within: radius)
        # the shared explosion visibility test: rays from the centre to the
        # object's bones, the fraction that reach it scaling the effect

    # Damage is applied ONLY to things that are neither ragdolls nor
    # walking creatures. A creature on its feet and a body already
    # simulated as a ragdoll take the PUSH but no damage.
    IF the object has a ragdoll shell THEN damage = 0
    ELSE IF the object is a living creature still walking THEN damage = 0
    ELSE damage = impulse

    IF impulse > small THEN
      FOR EACH affected bone
        send a hit event at the object: from this artefact, along the direction,
          with that damage, that impulse, at that bone, as a wound
```

**Invariants** — the falloff is quadratic and crosses zero at exactly the radius, which is
why no separate range test is needed. The line-of-sight factor is the shared explosion
routine, so a shockwave is blocked by walls in exactly the way an explosion is — a rebuild
must share that routine between the two or they will diverge visibly.

**Notes**

- **The per-bone loops never execute.** The two lists they drain — the affected bone
  identifiers and their positions — are declared empty at the top of the function and
  nothing ever fills them. As shipped, this artefact computes a full shockwave and sends no
  hits at all: the only visible effect is the particle burst. Whatever filled those lists
  was removed. A rebuild reproducing the shipped behaviour does nothing; a rebuild
  reproducing the *intent* sends one hit per object at its root bone. This is the single
  largest unrecoverable decision in these files.
- The factor of one hundred on the impulse is unexplained, and combines with the configured
  strike impulse to produce the final value.
- Two lines that would have disabled the artefact during the sweep are commented out. They
  were presumably an attempt to stop the artefact hitting itself; the nearby list's
  artefact exclusion does that instead.

## the touch sense and `net_Relcase`

**Contract** — membership is any object with a physics presence that is *not* an artefact.
The reference-release hook removes an object from the list when it is destroyed, using a
remove-then-erase over the whole list rather than a single find, so a duplicated entry is
also cleaned.

**Invariants** — implementing the release hook is mandatory for any object holding a list of
other objects across frames. Its sibling class
([`BastArtifact.cpp`](BastArtifact.cpp.md)) omits it and is wrong to.

**Notes** — the add path tests for a physics-shell holder and the remove path tests for a
plain game object, so an object that is the latter but not the former is searched for in a
list it was never added to. The search is unchecked and will run past the end. A rebuild
should test the same predicate on both paths.
