# src/xrGame/WeaponRevolver.cpp

> A revolver: a semi-automatic-feeling handgun whose reload animation depends on how many rounds are still in the cylinder.

**Needs** — [`WeaponRevolver.h`](WeaponRevolver.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: animation selection on the firing and reload paths.

## Purpose

A revolver's reload is visibly different depending on how many spent cases must come out,
and its cylinder is visibly empty or not at rest. So the class is an animation table:
six reload variants indexed by the rounds remaining, and the same empty/loaded split over
the rest poses that [`WeaponPistol.cpp`](WeaponPistol.cpp.md) has.

It is a sibling of the pistol rather than a subclass, which costs a second copy of the
empty-variant table. A rebuild should make "has an empty rest pose" a property of a
weapon rather than a class.

## State

```text
RECORD Revolver EXTENDS CustomPistol
  close_sound : SoundHandle      # the cylinder swinging shut
```

## `PlayAnimReload` — the count-indexed table

**Contract** — the reload animation is chosen by how many live rounds remain in the
cylinder, so that the animation ejects and loads the right number.

```text
FUNCTION play_reload_animation(revolver)
  MATCH revolver.ammo_elapsed
    1 -> "anm_reload_1"
    2 -> "anm_reload_2"
    3 -> "anm_reload_3"
    4 -> "anm_reload_4"
    5 -> "anm_reload_5"
    otherwise -> "anm_reload"        # covers 0 and a full cylinder
```

**Invariants** — the index is the count *before* the reload transfers anything, which is
correct because the transfer happens when the animation ends. Note the table stops at
five: a six-round revolver reloading from full falls into the default, which is also the
empty case. That collision is the one place a rebuild would want a sixth entry.

**Notes** — unlike every other animation in this hierarchy, these six probe only the
newer name. They were added after the older data set was frozen, so no older name exists.

## The empty-variant table

**Contract** — the same seven slots as the pistol, selected on whether the cylinder is
empty: draw, inspect, sprint idle, moving idle, rest idle, aim idle and holster. As with
the pistol, holstering an empty revolver also plays the cylinder-close sound.

## `PlayAnimShoot`

**Contract** — a distinct animation for the last round, on the same "will there be a
round after this one" test the pistol uses.

## `AllowFireWhileWorking`

**Contract** — true. A revolver accepts the next trigger pull while the previous shot's
animation is still playing, bounded by the shot clock and by the deferred trigger
release inherited from [`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md).

## Sound

**Contract** — one extra sound, the cylinder closing, tagged with the recharging sound
class. Re-anchored at the muzzle with the inherited set each frame.

## Pass-throughs

`net_Destroy`, `OnH_B_Chield`, `switch2_Reload`, `OnAnimationEnd` and `OnShot` all
forward unchanged. The last carries a comment explaining why: keeping an override that
merely forwards is cheaper to maintain than a copy that must track the base. A rebuild
should delete all five.
