# src/xrGame/CustomMonster_VCPU.cpp

> Turning "look that way" into an actual pose: converts a direction into a yaw/pitch pair, and advances the creature's body rotation toward its target at a bounded rate.

**Needs** — [`CustomMonster.h`](CustomMonster.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`CustomMonster_inline.h`](CustomMonster_inline.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: trigonometry and angle interpolation, once per thinking tick

## Purpose

Two small routines split out of [`CustomMonster.cpp`](CustomMonster.cpp.md) — the file
name suggests they were once meant for a vector coprocessor and the split is now arbitrary.
Both concern the same thing: the creature's *look*, which in this engine is a pair of
angles carried separately from the creature's position and turned into a transform each
tick.

## State

`Stateless.` Both routines operate on the movement manager's body rotation record.

## `mk_rotation`

**Contract** — converts a direction vector into a yaw and a pitch, with yaw measured as a
full turn rather than a half. Writes into a caller-supplied rotation; leaves roll alone.
The input direction is normalized in place as a side effect.

```text
FUNCTION direction_to_rotation(direction, out rotation)
  flat = (direction.x, 0, direction.z) normalized, or zero if degenerate
  clamp each component of flat just inside [-1, 1]     # see Notes

  # arccos alone only covers half a turn; the sign of x picks the other half
  IF flat.x >= 0 THEN rotation.yaw = arccos(flat.z)
  ELSE                rotation.yaw = 2*pi - arccos(flat.z)

  direction = direction normalized, or zero if degenerate
  rotation.pitch = -arcsin(direction.y)
```

**Invariants** — yaw is derived from the direction flattened into the horizontal plane and
pitch from the full direction, so a steeply inclined direction still yields a meaningful
facing. Pitch is negated: the engine's pitch convention is the opposite sign from the
mathematical one, the same handedness mismatch that appears at every angle boundary in
[`CustomMonster.cpp`](CustomMonster.cpp.md).

**Notes**

- Clamping the components just inside unit magnitude guards the arccos against a
  normalized vector whose component rounds fractionally past one — which does happen at
  single precision and produces a not-a-number that then poisons the creature's transform.
  A rebuild needs the guard, not necessarily the exact epsilon.
- All three components are clamped although only two are used; the vertical is clamped for
  nothing.

## `Exec_Look`

**Contract** — one tick of turning. Normalizes all four stored angles into the signed
half-turn range, moves the current yaw and pitch toward their targets at bounded rates, and
rebuilds the creature's transform from the resulting angles — preserving its position.
Does nothing at all when an animation currently owns the transform.

```text
FUNCTION look_tick(dt)
  IF an animation is driving the transform THEN RETURN

  normalize current.yaw, current.pitch, target.yaw, target.pitch
    into the signed half-turn range        # so the turn takes the short way round

  pitch_rate = the species' own pitch rate, derived from the body's turn rate
  turn current.yaw   toward target.yaw   at the body rate, capped at dt
  turn current.pitch toward target.pitch at the pitch rate, capped at dt

  saved_position = position
  rebuild the transform from (-last_model_yaw, -last_torso_pitch, 0)
  position = saved_position
```

**Invariants**

- the early exit is the invariant from
  [`CustomMonster.cpp`](CustomMonster.cpp.md): while an animation is moving the creature,
  nothing else may write the transform.
- the turn rate is *capped*, not scaled: the bounded lerp snaps exactly onto the target
  when the remaining difference is less than this tick's allowance, so a creature never
  overshoots and never oscillates around its target. That is
  [`angle_lerp_bounds`](CustomMonster_inline.h.md).
- pitch turns at a separately derived rate, so a species can be made to pitch its head
  faster or slower than it yaws its body.

**Notes** — the transform is rebuilt from the *interpolated network state's* angles, not
from the body rotation this function just advanced. The two are the same value one tick
later, so the creature's visible facing lags its intended facing by exactly one thinking
tick. Whether that is deliberate is not recorded.
