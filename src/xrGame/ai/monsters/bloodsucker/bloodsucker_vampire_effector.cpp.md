# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_effector.cpp

> What being fed on looks like: the screen pulses twice while the picture desaturates, and the camera is dragged to within a hand's breadth of the creature's face and held there, shivering.

**Needs** — [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md)
**Used by** — [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md)
**Tier floor** — T2: per-frame interpolation arithmetic over the camera basis and a colour-grading record; it hands its result to the camera stack rather than to a device

## Purpose

The bloodsucker's feed is the one attack in the game that takes the camera away from the player, and these two effectors are the whole of that experience. They are written as *timed* effectors — each is created with a lifetime, ticks itself down, and reports exhaustion — because the feed's length is decided by the behaviour, not by them.

Both run on the victim's camera, so they are consequences of the creature rather than parts of it. They live beside the creature because nothing else creates them.

## State

```text
RECORD VampirePostProcessEffector
  target_grade : colour-grading record    # the fully-applied look, supplied by the caller
  total_time   : real                     # the lifetime it was created with
  remaining    : real                     # inherited; counts down

RECORD VampireCameraEffector
  total_time  : real
  remaining   : real                      # inherited; counts down
  travel      : real                      # how far the camera must move
  direction   : vector (unit)             # which way it must move
  wobble_now    : vector of three angles  # current angular offset
  wobble_target : vector of three angles  # where each angle is heading
```

**Invariant** on the camera effector: `travel` and `direction` are computed once at construction from the victim's and the creature's positions and never recomputed. The camera therefore follows a straight line fixed at the moment the feed began, and does not track the creature if it moves.

## `VampirePostProcessEffector`

**Contract** — each frame, compute how far through its lifetime it is and blend from the neutral look toward the supplied one by a factor derived from that fraction. Reports "still alive" always; the camera stack retires it when its lifetime runs out.

```text
FUNCTION process(current_look)
  t = (total_time - remaining) / total_time     # 0 at the start, 1 at the end

  IF t < 0.2
    factor = 0.75 * t / 0.2                     # ramp in over the first fifth
  ELSE IF t > 0.8
    factor = 0.75 * (1 - t) / 0.2               # ramp out over the last fifth
  ELSE
    factor = 0.5 + 0.25 * sine(t mapped across two full turns)

  clamp factor to [0.01, 1]
  current_look = blend(neutral, target_grade, factor)
```

**Notes** — the middle section is two complete oscillations between 25 and 75 percent applied, whatever the feed's length. Reading it as a heartbeat is the obvious interpretation and matches the ramp shape, but nothing in the source says so; the count of two is not derived from anything.

The ramps reach 0.75 while the oscillation centres on 0.5, so the effect is at its strongest a fifth of the way in and again a fifth from the end, not in the middle. That is deliberate enough to be worth preserving — the blend is *weakest* at the moment the feed is longest established.

The lower clamp of 0.01 rather than 0 means the neutral look is never quite reached during the effect. Nothing depends on the difference.

The normalisation applied to the oscillation's input is written in a form that simplifies to dividing by one — the two corrections cancel. The oscillation therefore spans the whole lifetime rather than only the middle three fifths, which is almost certainly not what the expression was reaching for. A rebuild should decide deliberately which it wants.

## `VampireCameraEffector`

**Contract** — constructed with a lifetime, the camera's position and the creature's. It computes the distance between them and how far that is from the ideal feeding distance of 0.3 units, and the direction that closes the gap — noting that a camera *closer* than ideal is pushed away rather than pulled in. Each frame it advances the camera along that line, adds a wandering angular offset, and hands back the modified camera basis. It reports exhaustion when its lifetime runs out, which is how the camera stack knows to drop it.

**Invariants** — the camera returns to where it started: the displacement follows a semicircle that is zero at both ends, and the wobble is steered back to zero over the final fifth of the lifetime. A feed that is interrupted still ends with the camera where the player left it.

```text
FUNCTION process_camera(basis)
  remaining = remaining - frame_time
  IF remaining < 0  RETURN exhausted

  t = 1 - remaining / total_time              # 0 at the start, 1 at the end

  # displacement follows the upper half of a circle: zero at t=0 and t=1,
  # maximum at the midpoint
  offset = travel * sqrt(0.25 - (t - 0.5) * (t - 0.5))
  basis.position = basis.position + direction * offset

  IF the last fifth of the lifetime
    steer all three wobble angles toward zero at a rate that reaches
    zero exactly when the lifetime does
  ELSE
    FOR EACH of the three angles
      slew it toward its target at the fixed wobble rate
      IF it arrives, pick a new target uniformly within ten degrees

  rotate basis by the three wobble angles
  RETURN alive
```

**Notes** — the semicircular displacement profile is the load-bearing choice. Its derivative is infinite at both ends, so the camera leaves the player's control abruptly and snaps back the same way, while spending most of the feed near the creature's face and barely moving. A disabled triangular profile sits beside it in the source, which would have given constant speed out and back; the semicircle was chosen over it. Reproducing the triangle instead gives a noticeably gentler, less violent grab.

The wobble is three independent angles each drifting to a fresh random target within ten degrees at a fixed slew rate, which is why the shot never repeats and never settles. The return is handled by a different rule from the drift — the rate is recomputed each frame as "current offset divided by remaining time", so however far the angles have wandered they arrive at zero exactly when the effect ends. The small constant added to the remaining time is there only so the last frame's division stays finite.

The "ideal distance" of 0.3 units and the ten-degree wobble bound are fixed in code, not configured. Nothing in the source derives either; 0.3 reads as "close enough that the creature's head fills the frame" but that is a reading, not a record.
