# src/xrGame/weaponBM16.h

> Declares the double-barrelled shotgun: the shotgun behaviour with every animation selector overridden to branch on shells remaining.

**Needs** — [`weaponBM16.cpp`](weaponBM16.cpp.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`weaponBM16.cpp`](weaponBM16.cpp.md)
**Tier floor** — T3: a specialization that overrides presentation hooks only

## Purpose

Declares the surface implemented in [`weaponBM16.cpp`](weaponBM16.cpp.md). It adds no
state, no new behaviour and no new public operations to the shotgun it specializes: every
member it declares is an override of an animation or sound hook. That is the shape a rebuild
should preserve — a weapon subclass here is a *presentation* specialization, and everything
that decides what the weapon does lives in the base class and in the configuration section.

Exported units:

- `Load(section)` — base load plus the single-shell reload sound.
- `PlayAnimShoot`, `PlayAnimReload`, `PlayAnimIdle`, `PlayAnimIdleMoving`,
  `PlayAnimIdleSprint`, `PlayAnimShow`, `PlayAnimHide`, `PlayAnimBore` — the eight
  animation selectors, each branching on shells remaining.
- `PlayReloadSound` — one-shell versus two-shell reload sound.
- A script registration hook, which exports the class to the script layer with the *shotgun*
  as its declared base, so scripts see it as a shotgun with no added surface.

## State

`Stateless.`
