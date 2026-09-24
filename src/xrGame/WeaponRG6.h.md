# src/xrGame/WeaponRG6.h

> Declares the revolving grenade launcher implemented in [`WeaponRG6.cpp`](WeaponRG6.cpp.md).

**Needs** — [`RocketLauncher.h`](RocketLauncher.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md)
**Used by** — [`WeaponRG6.cpp`](WeaponRG6.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponRG6`, which combines two capabilities: spawning and launching real
grenade objects, and a shell-at-a-time magazine. Substance is in
[`WeaponRG6.cpp`](WeaponRG6.cpp.md).

The declaration's own content is the combination itself. Every entry point below has to
say which half it is delegating to, which is the cost of expressing "has a launcher" and
"has a shell-fed magazine" as two inheritances rather than two members.

Exported units:

- `net_Spawn` — after the magazine is restored, spawn one grenade per loaded round.
- `Load` — load both halves' parameters.
- `FireStart` — the ballistic solve and the launch; bypasses the bullet path entirely.
- `AddCartridge` — push a shell and spawn the grenade behind it.
- `OnEvent` — grenade ownership taken, rejected or launched.

Registered to the script virtual machine as a subclass of the shotgun.
