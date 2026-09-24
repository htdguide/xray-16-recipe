# src/xrGame/PhysicsGamePars.cpp

> Holds the speed thresholds at which a physics collision becomes audible, visible as a decal, or worth spawning particles for.

**Needs** — [`PhysicsGamePars.h`](PhysicsGamePars.h.md)
**Used by** — reached through its declarations in [`PhysicsGamePars.h`](PhysicsGamePars.h.md); callers name that, not this file.
**Tier floor** — T3: a handful of tuned constants and one configuration read

## Purpose

A collision between two physics bodies produces up to three cosmetic effects: a sound, a
wallmark and a particle burst. Each has its own speed threshold, so that resting contacts
and slow scrapes are silent while a real impact is loud. This file is where those
thresholds live, separated from the collision code that consults them precisely so that
the numbers can be found and retuned in one place.

Two threshold sets exist, and the split is the load-bearing decision: bodies that are
*props* and bodies that are *characters* respond at very different speeds. A character
ragdoll brushing scenery at walking pace would otherwise emit a collision sound on every
step, so its thresholds are raised — roughly double for sound, four times for particles,
three times for wallmarks.

## State

```text
# tuned thresholds, in world units per second of relative approach speed
RECORD EffectThresholds                 # for generic physics props
  sound      : real = 10
  particles  : real = 15
  wallmark   : real = 30

RECORD CharacterEffectThresholds        # for character bodies and ragdolls
  sound      : real = 20
  particles  : real = 60
  wallmark   : real = 100

# global, read from configuration at startup
collide_volume_min : real     # gain at the sound threshold
collide_volume_max : real     # gain at and above saturation
                              # invariant: min <= max, or loudness inverts with speed
```

The two threshold sets are compile-time constants; the two volumes are not, because they
are per-game mixing decisions that ship in the configuration.

## `LoadPhysicsGameParams`

**Contract** — reads the minimum and maximum collision-sound gain from the configuration's
sound section and stores them in the two globals. Called once during game bring-up, before
any level loads. No error case: a missing key is a fatal configuration error handled by
the configuration reader, not here.

**Notes** — the globals are mutable process state reached by name from the collision
callback. In a rebuild this is a small immutable settings record passed to, or owned by,
the physics-effects component; nothing here needs to be global except by historical
accident.

A commented-out third parameter scaled collision damage by a squared factor. It is gone
from the live path — collision damage is computed elsewhere — and a rebuild should not
reintroduce it.
