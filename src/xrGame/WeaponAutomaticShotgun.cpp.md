# src/xrGame/WeaponAutomaticShotgun.cpp

> An automatic shotgun: the shell-at-a-time reload of [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md), grafted onto full-automatic fire instead of semi-automatic.

**Needs** — [`WeaponAutomaticShotgun.h`](WeaponAutomaticShotgun.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Inventory.h`](Inventory.h.md)
**Used by** — reached through its declarations in [`WeaponAutomaticShotgun.h`](WeaponAutomaticShotgun.h.md); callers name that, not this file.
**Tier floor** — T2: a sub-state machine nested inside the weapon state machine.

## Purpose

An automatic shotgun needs two things that no single existing class provides: the burst
firing of [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) and the shell-at-a-time reload
of [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md). The shotgun's reload lives below the
semi-automatic behaviour, so it cannot be inherited without also inheriting one-shot
fire.

The resolution chosen was **to copy the entire reload sub-state machine**. This file is,
line for line, the reload half of the shotgun with a different base class. Every
procedure, every guard, every sound name and every invariant described in
[`WeaponShotgun.cpp`](WeaponShotgun.cpp.md) — the three phases, the fire-to-abort rule,
the type-switching availability predicate, the shell transfer on animation end, the
per-shell network replication — applies here unchanged. Read that page; it is this page.

**A rebuild should not copy it.** The reload sequence is a behaviour, not a class: factor
it into something a weapon *has* rather than something a weapon *is*, and give both
shotguns one instance of it. The duplication is the single clearest piece of accidental
structure in the weapon hierarchy.

## State

Identical to the shotgun's: the authored tri-state-reload flag and three sounds, with the
phase itself held on the weapon base so the network message can carry it.

## The differences from `WeaponShotgun.cpp`

Three, all incidental:

1. **The base class.** Firing comes from the magazined weapon — bursts, fire modes, the
   whole automatic cadence — instead of the semi-automatic pistol. There is no
   `switch2_Fire` override here at all.
2. **Two animation names differ.** The older names for the open and close animations are
   `anim_open` and `anim_close` rather than `anim_open_weapon` and `anim_close_weapon`.
   That is data, and both spellings ship.
3. **No `net_Destroy` override.** The shotgun has an empty one; this does not.

Everything else — including the running-total quirk in the availability predicate and the
shooting-class tagging of the reload sounds — is identical and is documented once, on the
shotgun's page.

## Script-visible ancestry

**Contract** — registered to the script virtual machine as a subclass of the *magazined
weapon*, not of the shotgun, matching its real ancestry. A script asking "is this a
shotgun" answers **false** for this weapon, and the shipped scripts depend on that.
