# src/xrGame/weaponBM16.cpp

> The double-barrelled shotgun: a weapon whose every animation is chosen by how many shells are still in it.

**Needs** — [`weaponBM16.h`](weaponBM16.h.md) · [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md)
**Used by** — [`weaponBM16.h`](weaponBM16.h.md)
**Tier floor** — T3: a table lookup from load state to animation name

## Purpose

Every other weapon plays one draw animation, one holster, one idle. This one plays three of
each, because the model has visible barrels and the player can see whether they are loaded.
The whole file is that: the load-state-to-animation mapping, plus the two-shell reload's
own branch.

It is a separate class rather than configuration because the *mapping* is per-weapon
structure, not a per-weapon number — a rebuild that wants it in data needs an authored
table of (state, shells remaining) to animation name, which the original never grew.

## State

`Stateless.` All state is the magazine the shotgun base class owns; this file only reads
its count.

## `Load`

**Contract** — takes the base shotgun load, then additionally binds one extra sound: the
single-shell reload. The base class already bound the two-shell reload sound under the
plain name, so the section must name both.

**Notes** — the extra sound is registered as belonging to the *shot* sound category, which
means it is attenuated and occluded like a gunshot and — more to the point — carries the
same AI-perception attributes into the `feel` layer. Reloading a shotgun is audible to
creatures the way firing it is. That is a deliberate choice about how loud reloading a
break-action weapon is, and a rebuild that files the reload under a quieter category changes
how easily a player is heard.

## `PlayReloadSound`

**Contract** — plays the single-shell reload sound when exactly one shell remains in the
magazine, and the two-shell one otherwise. Emitted at the weapon's last fire point, so the
sound comes from the muzzle rather than from the holder's origin.

## `PlayAnimShoot`

**Contract** — selects the firing animation by shells remaining *at the moment of firing*:
one shell means the second-barrel animation, two means the first-barrel animation. With
zero shells nothing is played at all — there is no shot to animate.

**Invariants** — the count is read before the shell is consumed, so "one remaining" is the
last shot, not the one after it.

## `PlayAnimShow`, `PlayAnimHide`, `PlayAnimBore`, `PlayAnimIdleMoving`, `PlayAnimIdleSprint`

**Contract** — five animation selectors with the identical shape: pick one of three
animations by the shell count (zero, one, two). Each names both a first-person animation
and a corresponding world-model animation; draw and holster are marked as one-shot, the
idles as looping.

**Notes** — The empty case falls back to the *generic* world-model animation name in every
one of these while the loaded cases use count-specific names. That asymmetry is in the
shipped data, not a bug here: the world model has no visibly-empty variant, only the
first-person view does.

## `PlayAnimIdle`

**Contract** — the idle selector, which is the only one with two axes: aimed or not, times
three shell counts, for six animations. Defers first to the base class's generic idle
attempt, and does nothing further if that consumed the request. Passes no owner for the
animation callback, unlike the other selectors, so no completion event is raised — an idle
loops until something else interrupts it.

## `PlayAnimReload`

**Contract** — selects between the one-shell and two-shell reload animation. Requires the
weapon to actually be in its reloading state. Plays the *one-shell* animation when either
only one shell is missing, or the holder does not have two rounds of the current type
available to load — and, additionally, only when the reload is not also an ammunition-type
change.

```text
FUNCTION play_reload_animation()
  both_available := holder has at least 2 rounds of the current ammunition type
  switching_type := a next ammunition type is pending AND it differs from the current one

  IF (magazine holds 1 OR NOT both_available) AND NOT switching_type
    play "reload one shell"
  ELSE
    play "reload both shells"
```

**Notes** — **A type change always animates as a full two-shell reload**, even when only one
shell is going in. That is not cosmetic: the animation's length is the reload's duration, so
swapping ammunition costs the full two-shell time regardless of how many shells are
involved. Reading it the other way round — the player is made to pay for changing their
mind — is probably the intent, but it is not recoverable from the source whether the
designers reasoned about it or whether the branch merely fell out of "the one-shell
animation does not have a place to show the new shell type".
