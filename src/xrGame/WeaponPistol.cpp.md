# src/xrGame/WeaponPistol.cpp

> A pistol: a semi-automatic weapon whose slide locks back when empty, so every animation has an empty variant.

**Needs** — [`WeaponPistol.h`](WeaponPistol.h.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: animation selection on the firing path.

## Purpose

One visual fact drives this entire file: a pistol's slide stays back after the last
round. That means the model has *two* rest poses, and every animation that starts or ends
at rest needs a second variant. The file is that variant table, plus the slide-closing
sound that the reload's tail needs.

## State

```text
RECORD Pistol EXTENDS CustomPistol
  close_sound : SoundHandle     # the slide running forward
```

## The empty-variant table

**Contract** — seven animation slots each pick between a loaded and an empty variant on
one test: is the magazine empty. Each variant, like every animation in this hierarchy,
probes two names — a newer and an older one — for data compatibility across the shipped
games.

| Slot | Loaded | Empty |
|---|---|---|
| draw | the base weapon's draw | `anm_show_empty` / `anim_draw_empty` |
| idle | the base weapon's idle | `anm_idle_empty` / `anim_empty` |
| idle while moving | the base's | `anm_idle_moving_empty` / `anim_empty` |
| idle while sprinting | the base's | `anm_idle_sprint_empty` / `anim_empty` |
| aim idle | the base's | `anm_idle_aim_empty` / `anim_empty` |
| inspect | the base's | `anm_bore_empty` / `anim_empty` |
| holster | the base's | `anm_hide_empty` / `anim_close` — **and** plays the slide-close sound |

The holster case is the only one with a side effect: holstering an empty pistol runs the
slide forward, so the sound plays with the animation.

**Notes** — the older name for every empty variant is the same string, so on the older
data set all six empty poses collapse to one clip. That is not a bug to correct; it is
what that data ships.

## `PlayAnimShoot` — the last round

**Contract** — firing uses a *different* animation for the last round in the magazine,
because that is the shot after which the slide locks back.

```text
FUNCTION play_shoot_animation(pistol)
  IF pistol.ammo_elapsed > 1 THEN play "anm_shots" / "anim_shoot"
  ELSE                            play "anm_shot_l" / "anim_shot_last"
```

**Invariants** — the test is against the count *before* the round is consumed, so
`> 1` means "there will still be a round after this one". Off by one here makes the slide
lock a shot early or a shot late, which is immediately visible.

## `PlayAnimIdle`

**Contract** — first offers the base hierarchy's chance to play a contextual idle (moving,
sprinting, aiming); only if none applies does it fall through to the loaded/empty rest
pose.

## `AllowFireWhileWorking`

**Contract** — **true**, unlike every other weapon in the hierarchy. A pistol accepts a
new trigger pull while its previous shot's animation is still running, which is what
makes rapid pistol fire possible at all — its animations are long relative to its cadence.
The rate is still bounded by the shot clock and by the deferred release in
[`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md).

## Sound

**Contract** — loads one extra sound, the slide closing, tagged with the recharging sound
class so the AI hears it as a reload rather than a shot. `UpdateSounds` re-anchors it at
the muzzle along with the inherited set.

## Pass-throughs

`net_Destroy`, `OnH_B_Chield`, `switch2_Reload` and `OnAnimationEnd` forward to the base
unchanged. They exist as override points that were never filled; a rebuild should delete
them.
