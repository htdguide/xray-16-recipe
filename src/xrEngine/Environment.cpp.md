# src/xrEngine/Environment.cpp

> The weather clock: which two authored time-of-day frames the world is currently between, how the transition to a weather effect is spliced in, and the per-frame interpolation that produces the sky, fog, sun and wind the renderer uses.

**Needs** — [`Environment.h`](Environment.h.md) · [`Environment_misc.cpp`](Environment_misc.cpp.md) · [`perlin.h`](perlin.h.md) · [`xrHemisphere.h`](xrHemisphere.h.md) · [`Rain.h`](Rain.h.md) · [`thunderbolt.h`](thunderbolt.h.md) · [`xr_efflensflare.h`](xr_efflensflare.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`Render.h`](Render.h.md) · [Configuration format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`Environment_misc.cpp`](Environment_misc.cpp.md)
**Tier floor** — T2: interpolation and list search over authored data. It reaches the device only through the renderer's environment interface.

## Purpose

Weather in this engine is not simulated. It is a list of *authored frames*, each stamped with a time of day, and the world at any instant is the linear blend of the two frames the clock currently sits between. Everything else here follows from that one decision: the search for the bracketing pair, the wrap across midnight, and the elaborate splice that grafts a weather effect (a storm) into the middle of a cycle and then grafts the cycle back.

This file owns the clock and the selection; [`Environment_misc.cpp`](Environment_misc.cpp.md) owns what a frame contains and how two frames blend.

## State

```text
RECORD Environment
  game_time        : real (seconds since midnight, 0 .. 86400)
  time_factor      : real            # game seconds per real second; default 12
  current[0..1]    : EnvFrame        # the bracketing pair; invariant: current[0] is before current[1] in cycle order
  current_env      : EnvFrameMixer   # the blend of the two; what everything downstream reads
  current_weather  : list<EnvFrame>  # the active cycle OR the active effect, sorted by time
  cycle_name       : text            # the cycle to return to when an effect ends
  weather_name     : text            # the list currently playing (cycle or effect)
  weather_cycles   : map<name, list<EnvFrame>>
  weather_effects  : map<name, list<EnvFrame>>
  modifiers        : list<EnvModifier>       # level-authored local overrides
  ambients         : list<EnvAmbient>
  effect_active    : bool
  effect_remaining : real (seconds)
  effect_end_pair  : EnvFrame[2]     # the cycle frames to resume into
  wind_strength    : real (0..1)     # gust envelope; driven by noise, not by the frames
  noise            : Noise1D
  clouds_mesh      : hemisphere vertices and indices
```

A day is 86400 seconds and every time value in this system is seconds-since-midnight. Game time advances at a configurable factor — twelve by default, so a full day-night cycle takes two real hours.

**Invariants** — Every cycle and every effect holds at least two frames; a single frame cannot be interpolated and load fails on one. Frames are kept sorted by time. The bracketing pair may *wrap*: when `current[0]`'s time is greater than `current[1]`'s, the pair spans midnight and every time comparison in this file must handle that case explicitly.

## `on_frame`

**Contract** — The per-frame entry point. Does nothing when no level is loaded. Runs the interpolation, advances the wind-gust envelope, and ticks the three visual weather effects. Does not advance the clock — the game does that, so that weather follows the simulation's notion of time including time acceleration and sleeping.

```text
FUNCTION on_frame()
  IF no level is loaded THEN RETURN
  interpolate()
  noise.frequency = gust_factor * 0.03
  wind_strength = clamp(noise.continuous(real_time_since_start) + 0.5, 0, 1)
  lens_flare.on_frame(current_env, time_factor)
  thunderbolt.on_frame(current_env)
  rain.on_frame()
```

**Notes** — Wind gusting is one-dimensional coherent noise sampled at *real* elapsed time, not game time, with two octaves and an amplitude of two thirds. The noise is biased by half and clamped into 0..1, so the envelope spends most of its time in the middle of the range and only occasionally reaches either extreme — which is what a gust feels like. The maximum frequency is 0.03 cycles per second, i.e. a gust period of about half a minute at full gustiness; the authored gust factor scales down from there.

**Notes** — The noise generator is seeded randomly at construction, so wind is not reproducible across runs. That is deliberate for a decorative effect, and it is a hazard for any rebuild that wants a deterministic replay — the demo-record path in this same module does want one.

## `interpolate`

**Contract** — Produces the current environment from the two bracketing frames, the weather modifiers near the camera, and the global visibility-distance multiplier. Ends a weather effect whose time has run out. Called once per frame, and also on demand after a forced weather change.

```text
FUNCTION interpolate()
  IF effect_active AND effect_remaining <= 0 THEN stop_effect()
  select_bracketing_pair(game_time)

  # Accumulate every level modifier the camera is inside, as one synthetic modifier.
  accumulated = zeroed modifier
  total_power = 0
  FOR EACH m IN modifiers
    total_power = total_power + accumulated.absorb(m, camera_position)

  f = time_weight(game_time, current[0].time, current[1].time)
  current_env.blend(current[0], current[1], f, accumulated, total_power)
  renderer.blend_environment_resources(current_env, current[0], current[1])
```

## `select_bracketing_pair`

**Contract** — Advances the bracketing pair so that the clock sits between them, handling the wrap across midnight. On the first call after a reset, both slots are empty and the pair is found from scratch; afterwards the pair is advanced by *sliding* — the upper frame becomes the lower and a new upper is found — because the clock moves forward continuously and a full search every frame would be wasted.

```text
FUNCTION select_bracketing_pair(t)
  IF both slots are empty THEN
    find the pair bracketing t and RETURN         # first start, or a forced weather change

  IF current[0].time > current[1].time THEN       # the pair wraps midnight
    advance = (t > current[1].time) AND (t < current[0].time)
  ELSE
    advance = (t > current[1].time)

  IF advance THEN
    current[0] = current[1]
    current[1] = first frame at or after t, wrapping to the first frame past the end
```

**Notes** — The wrapping test is an *and* of two conditions rather than an *or* precisely because the pair spans midnight: the clock is inside the pair when it is past the upper bound *or* before the lower one, so it has left the pair only when it is between them in ordinary order. Getting this backwards makes the weather freeze for most of the night.

**Notes** — Only one frame is advanced per call, so a clock that jumps by more than one frame's span — loading a save, a scripted time skip — lands on a pair that does not bracket it and drifts forward one frame per rendered frame until it catches up. The forced path exists to avoid that: a forced weather change clears both slots so the next selection is a full search.

## `select_frame` / `select_pair`

**Contract** — Binary search for the first frame at or after a given time. The single-frame form wraps to the first frame when the time is past the last one; the pair form additionally takes the previous frame as the lower bound, wrapping to the last frame when the time is before the first. Both rely on the list being sorted by time.

## `time_weight`

**Contract** — Where the clock sits between two times, as a fraction from zero to one, correct across midnight. Reports zero when the clock is outside the interval or the interval has zero length.

```text
FUNCTION time_diff(from, to) -> real
  IF from > to THEN RETURN (DAY_LENGTH - from) + to    # the interval wraps midnight
  RETURN to - from

FUNCTION time_weight(t, lo, hi) -> real
  length = time_diff(lo, hi)
  IF length is zero THEN RETURN 0
  inside = IF lo > hi THEN (t >= lo) OR (t <= hi) ELSE (t >= lo) AND (t <= hi)
  IF NOT inside THEN RETURN 0
  RETURN clamp(time_diff(lo, t) / length, 0, 1)
```

**Notes** — Reporting zero for a clock outside the interval, rather than extrapolating or clamping to the nearer end, means a desynchronized pair snaps the world to the *lower* frame rather than to the nearer one. Combined with the one-frame-per-call advance above, a large time jump produces a visible stutter rather than a smooth catch-up.

## `set_weather`

**Contract** — Switches the active weather cycle by name. An unknown name is reported and ignored rather than being fatal, because cycle names come from scripts. Forcing additionally invalidates the current pair so the new cycle takes effect this frame instead of at the next frame boundary; without forcing, the switch is deferred and the world crossfades into the new cycle through the pair it is already in. A weather effect in progress keeps playing — only the name to return to is updated.

**Notes** — An empty name is fatal, while an unknown name is not. The distinction is that an empty name means the caller has a bug and an unknown name means the data does.

## `set_weather_effect`

**Contract** — Splices a weather effect (a storm, a blowout) into the running cycle so that it fades in over a fixed transition, plays its authored sequence, and fades back into the cycle where the cycle would have been. Refuses if an effect is already playing. This is the most intricate algorithm in the file, and it works by *rewriting the effect's frame times* to sit on the current clock rather than by running a second interpolator.

```text
FUNCTION set_weather_effect(name)
  IF effect_active THEN RETURN refused
  previous_cycle = current_weather
  current_weather = weather_effects[name]

  # The transition is 5 game-seconds scaled by the time factor, i.e. 5 real seconds.
  rewind    = 5 * time_factor
  start     = game_time + rewind
  # Where we currently sit between the cycle's pair, handling a wrapped pair.
  current_weight, current_length = position and span of the cycle's active pair

  sort the effect's frames by their AUTHORED time
  first, second   = frames 0 and 1
  last_authored   = frame n-2
  tail            = frame n-1

  # Frame 0 becomes a copy of where the world is NOW, back-dated so that the
  # clock is already the right fraction into the [0,1] span. Frame 1 becomes a
  # copy of where the cycle was heading, stamped at the end of the transition.
  # The effect therefore fades in from exactly the current appearance.
  first.copy_appearance_of(cycle_current[0])
  first.time  = normalize(game_time - ((rewind / (cycle_current[1].time - game_time)) * current_length - rewind))
  second.copy_appearance_of(cycle_current[1])
  second.time = normalize(start)

  # Every authored effect frame is re-stamped relative to the transition end.
  FOR EACH frame f strictly between index 1 and the tail
    f.time = normalize(start + f.authored_time)

  # The tail is a copy of where the CYCLE will be when the effect finishes,
  # so the last transition fades back into the cycle seamlessly.
  effect_end_pair[0] = cycle frame at last_authored.time
  effect_end_pair[1] = cycle frame just after that
  tail.copy_appearance_of(effect_end_pair[0])
  tail.time = normalize(last_authored.time + rewind)

  effect_remaining = time_diff(game_time, tail.time)
  effect_active = true
  re-sort the effect's frames by their NEW time
  current[0], current[1] = first, second
```

**Notes** — Five game-seconds is the authored transition length, and it is multiplied by the time factor so that it is five *real* seconds at any time acceleration. Every other duration in this system is in game time; this one is not, because it is a presentation crossfade rather than a weather event.

**Notes** — The back-dating of frame 0 is the subtle part. Copying the current appearance into a frame stamped at the current clock would give a blend weight of zero and a hard cut; stamping it earlier by the amount that reproduces the current weight makes the first interpolation step continue exactly where the cycle's was. The expression is a proportion: the remaining cycle span is scaled by the ratio of the transition to the cycle's remaining time.

**Notes** — An effect list is padded at load with two synthetic frames, one at midnight and one at the end of the day, precisely so that indices 0, 1, n-2 and n-1 always exist and can be overwritten here without touching authored data. Those two frames are marked not-to-be-saved.

**Notes** — Frames are sorted twice, by two different keys: by *authored* time to identify the head and tail slots, then by *current* time once the stamps have been rewritten. That is why a frame carries both times.

## `start_weather_effect_from_time`

**Contract** — Starts an effect and then shifts every one of its frames so that the effect is already partly elapsed. Used when restoring a save taken during a storm, so the storm resumes where it was rather than restarting.

## `stop_weather_effect`

**Contract** — Ends the effect, restores the cycle by name without forcing, and installs the pair the splice recorded at the start. Because that pair was taken from the cycle at the time the effect would end, the world resumes the cycle without a jump.

## `set_game_time` / `change_game_time`

**Contract** — Setting installs an absolute clock and a new time factor, and shortens any running effect by the amount of time skipped so an effect does not survive a time jump. Changing advances the clock by a delta and wraps it into the day. Both are driven by the game, not by the engine's frame loop.

## `split_time`

**Contract** — Converts the clock to hours, minutes and seconds, taking the value modulo a day first so that an out-of-range clock is displayed rather than rejected.

## `invalidate`

**Contract** — Clears the bracketing pair, cancels any running effect, and drops the three per-frame references (ambient set, lens flare, thunderbolt collection) from the blended environment. Called on a forced weather change and after a device reset. The three drops are marked in the original as a hack: they exist because the blended environment holds pointers into frames that are about to be destroyed or reloaded, and nothing else clears them.
