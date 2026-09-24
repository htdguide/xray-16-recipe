# src/xrGame/ai/monsters/ai_monster_effector.cpp

> The player's-eye consequences of being attacked by a creature: a post-process envelope and a camera shake.

**Needs** — [`ai_monster_effector.h`](ai_monster_effector.h.md) · [`ActorEffector.h`](../../ActorEffector.h.md)
**Used by** — [`ai_monster_effector.h`](ai_monster_effector.h.md)
**Tier floor** — T2: per-frame interpolation of a parameter block and of a camera basis

## Purpose

A creature's attack has a visible effect on the player beyond damage: the screen washes,
the camera lurches. Both are expressed as *effectors* — objects the engine's camera and
post-process chains keep alive for a lifetime and consult every frame. The parameters come
from the creature's configuration section
(see [`ai_monster_defs.h`](ai_monster_defs.h.md), the attack-effector record), so each
creature's blow feels different without any code difference.

Both effectors are lifetime-bounded and self-terminating; the chain drops them when they
report they are finished.

## State

```text
RECORD PostProcessEffector
  target       : PostProcessInfo   # the fully-applied state, from configuration
  total        : real (s)          # lifetime at construction
  attack       : real [0..1)       # fraction of the lifetime spent ramping up
  release      : real [0..1)       # fraction at which the ramp down begins
  strength     : real              # overall multiplier

RECORD HitCameraEffector
  total        : real (s)
  max_amplitude: real              # degrees at full strength, already scaled by power
  periods      : real              # oscillations over the lifetime
  power        : real
  divisors     : vec3              # per-axis frequency and amplitude spread, randomized
                                   #   once at construction
```

**Invariants** — `attack` must not be zero and `release` must not be one, or the two ramp
expressions divide by zero. Both default to one half when the caller passes zero, which
means the unconfigured shape is a symmetric triangle: ramp up over the first half, ramp
down over the second, with no hold.

## `post-process effector`

**Contract** — constructed with a target post-process state, a lifetime in seconds, and
optionally the attack and release fractions and a strength factor. Called every frame with
the accumulating post-process parameters; blends its target into them by the envelope's
current value and reports that it is still alive. The chain's own lifetime accounting ends
it.

```text
FUNCTION process(accumulated)
  elapsed = (total - remaining) / total        # 0 at birth, 1 at death

  IF elapsed < attack THEN
    factor = elapsed / attack                  # ramp up
  ELSE IF elapsed <= release THEN
    factor = 1                                 # hold at full
  ELSE
    factor = (1 - elapsed) / (1 - release)     # ramp down

  clamp factor into [0.01, 1]
  accumulated = lerp(identity, target, factor * strength)
```

**Invariants** — the floor of 0.01 rather than 0 means the effect never fully vanishes
while alive. That matters because the blend is toward identity from the *identity* state,
not from the accumulated state: a factor of exactly zero would be indistinguishable from
the effector not existing, and the clamp is cheaper than special-casing it.

**Notes** — the blend overwrites the accumulated parameters rather than compounding into
them, so two creature effectors alive at once do not stack; the later one wins. Whether
that was intended is not recoverable.

The effector registers itself under the generic *hit* post-process category rather than a
creature-specific one, which means a creature attack and a bullet wound compete for the
same slot.

## `hit camera effector`

**Contract** — constructed with a lifetime, an amplitude in degrees, an oscillation count
and a power scalar. Called every frame with the camera's position, direction and up
vectors; rotates the direction and up by a decaying three-axis oscillation and writes them
back. Returns false once its lifetime is exhausted, which is what removes it.

```text
FUNCTION process(camera)
  remaining = remaining - frame_delta
  IF remaining < 0 THEN RETURN finished

  left = remaining / total                      # 1 at birth, 0 at death
  amplitude = max_amplitude in radians * left   # linear decay

  # three axes oscillate at different rates and amplitudes, set by the random
  # divisors chosen once at construction, so no two hits shake the same way
  yaw   =  amplitude / d.x * sin(periods * 2PI / d.x * (1 - left))
  pitch =  amplitude / d.y * cos(periods * 2PI / d.y * (1 - left))
  roll  =  amplitude / d.z * sin(periods * 2PI / d.z * (1 - left))

  build a rotation from (yaw, pitch, roll) and compose it onto the camera basis
  write back the rotated direction and up vectors
```

**Invariants** — the camera basis is rebuilt from the supplied direction and up vectors
with a cross product each frame; the effector never accumulates its own orientation, so its
output is a pure function of its remaining lifetime. Two of these alive at once compose
correctly for the same reason.

**Notes** — the divisors are drawn once at construction from 1–2 on one axis and 1–6 on the
other two, so the yaw axis oscillates roughly twice as hard and twice as fast as the
others. The asymmetry is what makes the shake read as a blow from the side rather than as a
rumble. The ranges are not derived anywhere.

Dividing the amplitude *by* the same divisor that divides the frequency means faster axes
shake less, which keeps the total angular velocity roughly equal across axes. That is
almost certainly why the same number appears in both places.
