# src/xrEngine/CameraBase.cpp

> Loads a camera's authored rotation limits and reports how close an angle is to one.

**Needs** — [`CameraBase.h`](CameraBase.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [Configuration format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — reached through its declarations in [`CameraBase.h`](CameraBase.h.md); callers name that, not this file.
**Tier floor** — T3: configuration reads and scalar arithmetic.

## Purpose

Two small decisions that are easy to get subtly wrong and are therefore worth stating: how an authored angle limit turns into "is this axis clamped at all", and what a limit test returns.

## `load`

**Contract** — Reads three authored values from a configuration section: a per-axis rotation speed, and low/high bounds for yaw and for pitch. Derives whether each axis is clamped, and centres the camera inside its range if it is. A missing key is a hard failure — the configuration is shipped data and a camera section that lacks these is a broken installation, not a case to default around.

```text
FUNCTION load(section)
  rot_speed = read_vector3(section, "rot_speed")
  lim_yaw   = read_vector2(section, "lim_yaw")
  lim_pitch = read_vector2(section, "lim_pitch")

  # An axis is clamped unless BOTH bounds are zero. A zero pair is the
  # authored idiom for "free rotation", which is why this is not a
  # low < high test: (0,0) and (-pi,pi) are both legal and mean opposite things.
  clamp_pitch = (lim_pitch.low != 0) OR (lim_pitch.high != 0)
  clamp_yaw   = (lim_yaw.low   != 0) OR (lim_yaw.high   != 0)

  # Start centred, so a clamped camera never begins already pressed against a stop.
  IF clamp_pitch THEN pitch = midpoint(lim_pitch)
  IF clamp_yaw   THEN yaw   = midpoint(lim_yaw)
```

Roll has limits in the record but none in the configuration: no shipped camera clamps roll, and the field exists so game code can set one directly.

## `check_limit_yaw` / `check_limit_pitch` / `check_limit_roll`

**Contract** — Map the current angle onto its authored range as a signed value: −1 at the low bound, +1 at the high bound, 0 at the centre, and beyond ±1 outside the range. Reports 0 when the axis is unclamped. Used by aim assistance and by the vehicle turret code to fade control authority as a stop is approached, which needs a smooth measure rather than a boolean.

```text
FUNCTION limit_position(limits, value) -> real
  RETURN (2*value - limits.low - limits.high) / (limits.high - limits.low)
```

**Notes** — All three report zero based on the **yaw** clamp flag, including the pitch and roll queries. On the shipped data this is invisible, because every camera section that authors a pitch limit also authors a yaw limit, so the flags agree. It is nevertheless wrong, and a rebuild should test each axis against its own flag: the correct reading of the intent is "report the position within the range when this axis has a range". Note also that an unclamped axis divides by a zero-width range if the flag is ever true without bounds — the flag derivation above is what prevents it.
