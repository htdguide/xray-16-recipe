# src/xrGame/EffectorShot.cpp

> Weapon recoil as a camera offset: each shot kicks the aim up and sideways by a randomized amount, the kick accumulates over a burst, and it relaxes back at a configured rate.

**Needs** — [`EffectorShot.h`](EffectorShot.h.md) · [`Weapon.h`](Weapon.h.md) · [`CameraRecoil.h`](CameraRecoil.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`Actor.h`](Actor.h.md)
**Used by** — reached through its declarations in [`EffectorShot.h`](EffectorShot.h.md); callers name that, not this file.
**Tier floor** — T2: two accumulating angles integrated per frame

## Purpose

The recoil model. It holds two angles — a vertical climb and a horizontal drift — that
every shot adds to and that decay back toward zero between shots. What makes it more than
a spring is a set of decisions that together produce the shipped weapons' distinctive
feel:

- **The kick grows through a burst.** Each successive shot without releasing the trigger
  adds a larger increment, so the first rounds are controllable and the tenth is not.
- **Horizontal drift is proportional to how far up the climb already is.** A weapon that
  has not climbed barely wanders; one at the top of its climb wanders hard. This is what
  makes the classic rising, widening recoil pattern rather than a straight vertical line.
- **The two axes relax together.** The horizontal relaxation rate is recomputed each frame
  so that it reaches zero at the same instant the vertical does, whatever the ratio
  between them, so the aim returns along a straight line rather than by an L-shaped path.
- **There are two return behaviours**, selected per weapon: a weapon in *return mode*
  pulls the aim back to where it started, and one not in return mode leaves the player's
  aim where the recoil put it and simply stops.

The file publishes both the accumulated offset and the per-frame *delta*, and the
distinction is load-bearing: the offset is what the camera is displaced by, and the delta
is what is added to the player's own aim angles so that recoil actually moves where the
weapon points rather than only where the camera looks.

## State

```text
RECORD CameraRecoil                # per-weapon tuning, read from the weapon's section
  Dispersion       : real   # vertical kick of the first shot, in radians
  DispersionInc    : real   # extra vertical kick per shot already fired in this burst
  DispersionFrac   : real   # in [0,1]: how much of the kick is deterministic.
                            # 1 = every shot kicks identically; 0 = uniformly random 0..kick
  MaxAngleVert     : real   # ceiling on accumulated climb
  MaxAngleHorz     : real   # ceiling on accumulated drift
  StepAngleHorz    : real   # drift scale, applied against the climb fraction
  RelaxSpeed       : real   # vertical return rate, radians per second
  RelaxSpeed_AI    : real   # the same for non-player shooters; not read here
  ReturnMode       : bool   # does the aim return to where it started
  StopReturn       : bool   # abandon the return if the player fights it; disabled, see Notes

RECORD RecoilState
  angle_vert, angle_horz            : real   # the accumulated offset
  prev_angle_vert, prev_angle_horz  : real   # last frame's, to derive the delta
  delta_vert, delta_horz            : real   # this frame's change, consumed by the aim
  shot_number                       : int    # shots fired so far in this burst
  active                            : bool   # the effector is contributing
  shot_end                          : bool   # the trigger was released
  single_shot                       : bool   # the weapon is in single-fire mode
  random_seed                       : int    # latched once; see SetRndSeed
```

Invariants: both angles are clamped to their configured ceilings. The delta pair is only
meaningful for the frame in which it was computed and must be consumed exactly once, or
recoil is applied twice or not at all.

## `Initialize` / `Reset`

**Contract** — `Initialize` copies in a weapon's tuning and clears the state; `Reset`
clears the state alone. Reset is what happens at the start of a burst, so the shot counter
and both angles begin from zero.

## `Shot`

**Contract** — called once per round fired. Reads how many rounds the weapon has fired in
this burst and how the fitted silencer scales both the base kick and the per-shot growth,
computes this shot's vertical kick, and applies it. A weapon reporting its first shot
resets the accumulator first, which is how a burst boundary is detected — the weapon's
shot counter, not a timer, defines a burst.

```text
FUNCTION shot(weapon)
  shot_number = weapon.shots_fired - 1
  IF shot_number <= 0 THEN
    shot_number = 0
    reset()                                  # new burst
  single_shot = weapon.fire_mode == 1

  kick = Dispersion    * weapon.silencer.dispersion_scale
       + DispersionInc * weapon.silencer.dispersion_growth_scale * shot_number
  apply_kick(kick)
```

**Notes** — the silencer scales the two dispersion terms independently, so a silencer can
be tuned to soften the first shot without changing how fast a burst climbs, or the
reverse.

## `Shot2` — applying one kick

**Contract** — the kick application, also callable directly by shooters that do not go
through a weapon. Mixes a deterministic and a random share of the kick into the vertical
angle, clamps it, then derives a horizontal step from how close the vertical is to its
ceiling. Marks the effector active and the burst not ended.

```text
FUNCTION apply_kick(kick)
  # DispersionFrac splits the kick between "always this much" and "up to this much, signed"
  angle_vert = angle_vert + kick * (DispersionFrac
                                    + random(-1, +1) * (1 - DispersionFrac))
  clamp angle_vert to +/- MaxAngleVert

  IF angle_vert is exactly at the ceiling THEN
    angle_vert = angle_vert * random(0.96, 1.04)     # jitter the pin; see Notes

  climb_fraction = angle_vert / MaxAngleVert
  angle_horz = angle_horz + climb_fraction * random(-1, +1) * StepAngleHorz
  clamp angle_horz to +/- MaxAngleHorz

  active   = true
  shot_end = false
```

**Notes**

- The jitter applied when the vertical angle pins at its ceiling exists because a pinned
  angle is perfectly still, and a perfectly still camera in the middle of automatic fire
  looks broken. Four percent either way is enough to keep it alive. Note that the jitter
  can push the angle past the ceiling it was just clamped to; nothing corrects it before
  the next shot's clamp, so the ceiling is soft by a few percent by design.
- The random share is signed, so with a low deterministic fraction a shot can kick the
  muzzle *down*. That is deliberate — it is how a weapon is made to feel loose rather than
  merely strong.

## `Relax`

**Contract** — one frame of return. The vertical angle moves toward zero at the configured
rate and is snapped to exactly zero on crossing, which also deactivates the effector. The
horizontal rate is derived each frame as "however fast it must go to arrive at the same
time", so the two reach zero together.

```text
FUNCTION relax()
  time_to_zero    = |angle_vert| / RelaxSpeed
  horz_rate       = 0 IF time_to_zero is zero ELSE |angle_horz| / time_to_zero

  move angle_horz toward zero by horz_rate * frame_seconds
  move angle_vert toward zero by RelaxSpeed * frame_seconds
  IF angle_vert crossed zero THEN
    angle_vert = 0
    active     = false
```

**Notes** — the horizontal angle is *not* snapped or checked for crossing; it is only the
vertical that ends the relaxation. Because the rates are matched each frame they arrive
together to within a frame, and the residue is below what a player can see. A rebuild
should still zero both.

## `Update`

**Contract** — the once-per-frame tick. Relaxes if the weapon returns its aim; deactivates
without relaxing if it does not and the trigger has been released; then latches this
frame's deltas from the change in each angle. Must run exactly once per frame, because the
deltas are differences against the previous call.

```text
FUNCTION update()
  IF active AND ReturnMode THEN relax()
  IF NOT ReturnMode AND shot_end AND NOT single_shot THEN active = false

  delta_vert = angle_vert - prev_angle_vert
  delta_horz = angle_horz - prev_angle_horz
  prev_angle_vert = angle_vert
  prev_angle_horz = angle_horz
```

**Notes** — in non-return mode the angles simply stay where the last shot left them and
the deltas go to zero. The accumulated offset is never unwound; it is the player's job to
pull the aim back down. The single-shot exemption keeps a single-fire weapon "active" so
that its one kick is delivered even though the trigger is already released.

## `GetDeltaAngle` / `GetLastDelta`

**Contract** — the two published outputs, both negated because the recoil angles are stored
as "how far the muzzle has climbed" and the consumers want "how far to move the view".
`GetDeltaAngle` yields the whole accumulated offset, used to displace the camera;
`GetLastDelta` yields this frame's change, used to move the aim itself. Roll is always
zero — recoil never rolls the view.

## `ChangeHP`

**Contract** — applies this frame's delta directly to a pitch and yaw pair, subtracting so
that a positive climb raises the aim. This is the path by which recoil moves where the
weapon actually points, as opposed to merely where the camera looks.

## `SetRndSeed`

**Contract** — seeds the effector's own random source, but **only the first time it is
called**; later calls are ignored. What it actually seeds with is the current frame
number, not the value passed in.

**Notes** — this is a deliberate defeat of a multiplayer feature, preserved as-is. The
argument exists so that a server can hand every client the same seed and have their recoil
patterns agree; latching on first call and then ignoring the argument in favour of the
frame counter makes recoil locally random and unsynchronized. The original's intended
behaviour survives only as a disabled line. A rebuild that wants deterministic recoil must
use the passed seed and re-seed when told to.

## `CCameraShotEffector`

**Contract** — the recoil model wearing a camera-effector face, so it can sit in the
camera chain and be ticked by it. Its per-frame work is exactly the recoil update; it does
not itself perturb the camera, because the camera displacement is pulled by the character
through `GetDeltaAngle` at a different point in the frame. Declares an effectively
infinite lifetime — it is installed with the weapon and removed with it — and carries the
identifier of the weapon it belongs to so the character can tell whose recoil this is.

**Notes** — an effector that is ticked by the chain but does not use the chain's output is
a sign the split is arbitrary. A rebuild can drop the effector face and tick the recoil
model from the weapon.

## Could not recover

- `RelaxSpeed_AI` is loaded into the tuning record and never read anywhere in this file;
  it presumably once gave non-player shooters a different return rate.
- `StopReturn`, and the first-shot position it was to be compared against, survive only as
  commented-out code. The intent is legible — abandon the automatic return once the player
  has pulled the aim past where the burst began — but it does not run.
- `m_first_shot` is set and never consulted.
