# src/xrGame/WeaponSVD.cpp

> A semi-automatic sniper rifle: one shot per pull, and the weapon stays locked for the whole length of the shot animation.

**Needs** — [`WeaponSVD.h`](WeaponSVD.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md)
**Used by** — reached through its declarations in [`WeaponSVD.h`](WeaponSVD.h.md); callers name that, not this file.
**Tier floor** — T2: two overrides of the firing state machine.

## Purpose

A marksman's rifle differs from a pistol in one respect that matters to the state
machine: after each shot the weapon must be *unavailable* for the full cycling animation,
not merely rate-limited. This file is that one difference.

## State

`Stateless.`

## `switch2_Fire`

**Contract** — identical to the semi-automatic base's arming, plus one addition: the
weapon is marked **pending** on entering the fire state.

```text
FUNCTION switch_to_fire(rifle)
  rifle.fire_single_shot = true
  rifle.firing            = false
  rifle.pending           = true        # <- the difference
  rifle.shots_fired       = 0
  rifle.stopped_after_queue = false
```

Pending is checked first by every input handler in the hierarchy, so between the shot and
the end of its animation the rifle accepts nothing at all: no second shot, no reload, no
zoom change, no holster. That enforced beat is the weapon's character.

## `OnAnimationEnd`

**Contract** — clears pending when the fire animation finishes, then defers to the base
table (which moves the state to idle).

**Invariants** — pending is set on entry to fire and cleared on the fire animation's end.
The two must pair: if the animation is interrupted without its end callback the rifle
stays locked, which is why the hidden-state transition in
[`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) clears pending on the way to idle rather
than relying on the callback.

**Notes** — the source comment on the fire case reads "end of reload animation", copied
from the case above it. It is the fire animation.
