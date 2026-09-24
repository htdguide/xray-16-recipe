# src/xrEngine/Rain.cpp

> Rain as the player experiences it: drops born around the camera, each ray-traced once to find where it lands.

**Needs** — [`Rain.h`](Rain.h.md) · [`Environment.h`](Environment.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`Rain.h`](Rain.h.md)
**Tier floor** — T1: a per-frame ray budget against the collision database and a preallocated splash pool

## Purpose

Rain has to look like it falls on the world — drops must stop at a roof and splash on the
ground, not pass through — while costing a fixed amount regardless of how hard it is
raining. The design that achieves this is: drops exist only in a cylinder around the
camera, each drop is ray-traced against the world *once* at birth rather than tested each
frame, and its whole future (when it arrives, whether it splashes) is computed from that
one trace. A drop that reaches the end of its predicted life is re-born, so the population
is constant.

This file owns that logic and the ambient sound. It owns no drawing.

## State

```text
RECORD Drop                        # one falling drop
  position    : vector3            # spawn point, world space
  hit_point   : vector3            # where the single trace said it lands
  direction   : vector3            # unit; gravity plus wind plus a random cone
  speed       : real               # metres per second
  expire_time : int                # global ms at which this drop is reborn
  hit_time    : int                # global ms at which it splashes; see note
  uv_variant  : int                # 0 or 1: which of two drop textures

RECORD Splash                      # one impact effect
  next, prev : optional<Splash>    # intrusive links; see below
  xform      : matrix4             # placed at the impact, randomly spun about the vertical
  bounds     : sphere              # for culling
  time       : real                # seconds remaining

STATE
  drops    : list<Drop>            # sized by the renderer's desired count
  splashes : Splash[1000]          # fixed pool, never resized
  active   : optional<Splash>      # head of the in-use list
  idle     : optional<Splash>      # head of the free list
  state    : {IDLE, WORKING}
  ambient  : SoundSource           # looping rain bed

# invariant: every pool entry is on exactly one of the two lists
# invariant: drops never allocate after creation; a spent drop is rewritten in place
```

The tuning constants, and what each one is actually for:

```text
source_offset      = 40      # metres above the camera drops are born, and the sound's inner radius
max_distance       = 50      # = source_offset * 1.25; the trace length, so a drop
                             #   can land up to 10 m below the camera before giving up
drop_angle         = 3 deg   # random spread cone around the wind-tilted fall direction
drop_max_angle     = 10 deg  # maximum tilt from vertical at full wind
drop_max_wind_vel  = 20      # wind speed that produces full tilt
drop_speed_min     = 40      # metres/second
drop_speed_max     = 80
max_particles      = 1000    # splash pool size; a hard cap on simultaneous splashes
particles_time     = 0.3     # seconds a splash lives
```

`max_distance` being a multiple of `source_offset` rather than an independent number is the
only relationship among these that carries meaning: the trace must reach past the camera or
drops would vanish at eye level.

## `Born`

**Contract** — fills one drop. Decides a fall direction from the current wind, a spawn point
on a disc above the camera, a random speed, then performs the single world trace that
settles the drop's whole future. Called once per drop per cycle; this is the only place the
collision database is touched.

```text
FUNCTION born(drop, radius)
  # direction: straight down, tilted into the wind, plus a small random cone
  gust  = environment.wind_strength_factor / 10
  k     = clamp(environment.wind_velocity * gust / drop_max_wind_vel, 0, 1)
  pitch = drop_max_angle * k - 90 deg
  axis  = direction from (wind_heading, pitch)
  drop.direction = random direction within drop_angle of axis

  # spawn point: uniform on a disc of the given radius, lifted above the camera and
  # pushed back along the fall direction so the drop passes through the camera's disc
  angle = random in [0, 2pi)
  dist  = sqrt(random in [0,1)) * radius        # sqrt: uniform over area, not over radius
  drop.position = camera + (dist*cos(angle), source_offset, dist*sin(angle))
                         - drop.direction * source_offset

  drop.speed = random in [drop_speed_min, drop_speed_max]

  height = max_distance
  hit = ray_pick(drop.position, drop.direction, height, both static and dynamic)
  renew(drop, height, hit)
```

**Notes** — the square root on the radius is what makes drops uniformly dense across the
disc; sampling the radius linearly would crowd them at the camera. The back-offset along the
fall direction is why a tilted drop still crosses the camera's column rather than being
blown past it before it arrives.

## `RenewItem`

**Contract** — given the traced distance and whether anything was hit, sets the drop's two
timestamps and its impact point. Picks one of two texture variants at random so a field of
drops does not visibly repeat.

```text
FUNCTION renew(drop, height, hit)
  drop.uv_variant = random 0 or 1
  travel_ms = 1000 * height / drop.speed
  drop.expire_time = now_global_ms + travel_ms - frame_delta_ms
  IF hit
    drop.hit_time  = now_global_ms + travel_ms - frame_delta_ms      # splash on arrival
    drop.hit_point = drop.position + drop.direction * height
  ELSE
    drop.hit_time  = now_global_ms + 2 * travel_ms - frame_delta_ms  # never reached
    drop.hit_point = drop.position
```

**Notes** — when nothing was hit, the splash time is set to *twice* the drop's own life, so
it is always in the future when the drop is reborn and the splash never fires. That is the
"no impact" encoding: a single comparison in the drawing code handles both cases with no
flag. Both timestamps subtract the current frame's delta, which back-dates them by one
frame so a drop born this frame is already partway down and the field does not visibly
pulse when many drops are reborn together.

## `RayPick`

**Contract** — one trace against the level, excluding the entity the camera is attached to
so the player's own body does not catch every drop. Returns whether anything was hit and
narrows the distance to the nearest hit. This is the effect's entire cost against the
collision database and the reason the drop count is capped.

## `OnFrame`

**Contract** — runs once per frame from the persistent layer. Reads the current weather's
rain density, drives a two-state machine over the ambient sound, and smooths a *sky
visibility* factor that dims the sound indoors. Does nothing with no level loaded, and
nothing at all on a dedicated server.

```text
FUNCTION on_frame(rain)
  IF no level loaded OR dedicated server
    RETURN

  factor = environment.rain_density

  # sky visibility: the brightest of five of the six hemisphere-cube faces sampled at the
  # camera entity, low-pass filtered. Under a roof it falls, outdoors it rises.
  IF camera entity has a lighting cache
    hemi = max over faces {0,1,2,3,5} of its hemisphere cube
    t = clamp(frame_delta_seconds, 0.001, 1.0)
    sky_visibility = sky_visibility * (1 - t) + hemi * t

  IF state IS IDLE
    IF factor IS effectively zero
      RETURN
    state = WORKING
    ambient.play(looping) at the listener, ranged [source_offset, 2 * source_offset]
  ELSE IF state IS WORKING
    IF factor IS effectively zero
      state = IDLE
      ambient.stop()
      RETURN

  ambient.volume = max(0.1, factor) * sky_visibility
```

**Notes** — the sound is positioned at the origin of listener space and given a range, not
placed in the world: rain has no source, it is everywhere, and giving it a range is what
makes it fade rather than cut when the listener moves. The filter coefficient is the frame
delta itself, so the smoothing settles in about a second of real time regardless of frame
rate — a step response that is frame-rate independent, which matters because this factor is
audible.

The hemisphere cube's face 4 is skipped. The five that are read are the four sides and one
vertical; the omitted one is the downward face, which sees the ground and would report dark
outdoors.

The lighting-cache read makes the sound depend on the *camera entity's* lighting, not the
listener's position, so a player standing in a doorway hears what the doorway sees.

## `Hit`

**Contract** — requests a splash at a world position. Half of all requests are dropped at
random, and a request that finds the pool exhausted is dropped silently. The splash is
placed at the point, spun randomly about the vertical so repeated splashes do not look
stamped, and given its bounding sphere transformed into world space for culling.

```text
FUNCTION hit(rain, position)
  IF random 0 or 1 is not zero
    RETURN                                  # half the impacts get no splash
  p = allocate_splash()
  IF p IS none
    RETURN                                  # pool exhausted: silently drop
  p.time  = particles_time
  p.xform = rotation about vertical by random angle, translated to position
  p.bounds = the renderer's drop bounding sphere, transformed by p.xform
```

**Notes** — the fifty-percent rejection is a cost control disguised as art direction: it
halves the splash population for a difference nobody sees in a field of thousands of drops.

## The splash pool

**Contract** — a fixed array of splashes threaded as two intrusive doubly-linked lists,
active and idle, with no allocation after creation. Allocating unlinks the idle head and
links it into the active list; freeing does the reverse. Allocation returns nothing when
the idle list is empty rather than growing.

```text
FUNCTION allocate() -> optional<Splash>
  IF idle IS none
    RETURN none
  p = idle
  unlink p FROM idle
  link p INTO active
  RETURN p
```

**Notes** — intrusive links rather than indices because the renderer walks the active list
directly. A rebuild may use a free-list of indices and a dense active array, which is
better for the renderer's traversal; the load-bearing part is only that the pool is fixed
and exhaustion is non-fatal.

## `Render`

**Contract** — pure delegation to the renderer's rain drawer, passing the whole effect. Does
nothing with no level loaded. The drawer reads the drop field and the active splash list
directly; it also owns the drop geometry's dimensions, the desired drop count and the
spawn radius, which is why several constants in this file are marked in the source as
duplicated on the renderer side.

**Notes** — that duplication is a real hazard: the drop count and spawn radius exist in two
places and must agree, or drops are born outside the radius the drawer assumes. A rebuild
should put both on one side of the boundary — the effect side, since it is the one that
uses them to place drops.
