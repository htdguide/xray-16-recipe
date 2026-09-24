# src/xrGame/Level_Bullet_Manager.cpp

> Every bullet and fragment in flight, simulated centrally as a ballistic trajectory swept against the world, with hits deferred to a frame boundary.

**Needs** — [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`Level.h`](Level.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Tracer.h`](Tracer.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`mt_config.h`](mt_config.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md); callers name that, not this file.
**Tier floor** — T1: a per-frame sweep over a hot array, run on a worker thread, with hard latency limits

## Purpose

A bullet in this engine is **not an object**. It is a record in one flat array owned by the
level, advanced by one central routine, with no scheduler registration, no network identity
and no lifecycle. That is the central decision of the file and it is what makes hundreds of
simultaneous projectiles affordable.

The file does four things: model a ballistic trajectory with air resistance and gravity,
find where that curved trajectory first meets the world, *defer* what that means until a safe
point in the frame, and draw tracers.

Hit resolution itself — what a hit does to a material, an armour value or a bone — lives in
[`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md). This file
decides *where* and *when*; that one decides *what*.

## State

```text
RECORD Bullet
  # identity and provenance
  id, parent_id, weapon_id, target_id : int (16-bit each)
  init_frame_num  : int      # the frame it was created on; it may not draw on that frame
  born_time       : int (ms)
  # kinematics — the trajectory is a closed-form function of these three plus time
  start_position  : position
  start_velocity  : vector
  life_time       : real     # seconds since the current trajectory segment began
  bullet_pos      : position # the evaluated position now
  dir             : direction
  speed           : real
  fly_dist        : real     # total distance travelled, across all segments
  change_trajectory_count : int   # segments so far this frame; capped at 32
  tracer_start_position   : position   # where it was at the start of this frame
  # cartridge-derived constants
  hit_param       : { power, impulse }
  max_speed, max_dist, air_resistance, armor_piercing,
  material_piercing, wallmark_size : real
  bullet_material_idx : int
  hit_type        : enum
  colour_id       : int (8-bit)
  flags           : { ricochet_was, explosive, allow_tracer, allow_ricochet,
                      allow_sendhit, aim_bullet, magnetic_beam }
  whine_sound, material_sound : sound handles

RECORD BulletManager
  bullets          : list<Bullet>   # the working set
  bullets_rendered : list<Bullet>   # a full copy, taken at end of frame, for drawing
  events           : list<Event>    # deferred hits and removals
  whine_sounds     : list<sound>
  explode_particles: list<text>
  tracers          : tracer renderer
  # tuning, all from configuration
  gravity_const, air_resistance_k, min_bullet_speed,
  collision_energy_min, collision_energy_max, hp_max_dist,
  tracer_width, tracer_length_min, tracer_length_max : real
```

**Invariants**
- The trajectory is a **closed-form function of time** from the segment's start position and
  velocity. There is no per-step integration, so the same bullet lands in the same place
  regardless of the frame rate. That is a hard requirement — the client and the server must
  agree on where a bullet went.
- A bullet's `life_time` is measured from the start of its *current* segment, and a segment
  restarts at every ricochet or penetration. Total travel is tracked separately.
- The removal event carries the bullet's **array index** in the field otherwise used for the
  target material. That overload is why the working set must be iterated in reverse: removal
  is by swap-with-last, which invalidates indices above the removed one but not below.
- Hits are never applied where they are found. Finding happens during the sweep, possibly on
  a worker thread; applying happens at the top of the next frame on the main thread. Nothing
  in the sweep may touch an object's state.
- A bullet may not be drawn on the frame it was created — the flag is a frame-number
  comparison — because its tracer would be a zero-length segment at the muzzle.

## The ballistic model

**Contract** — the trajectory is in two phases, joined at a fixed time, and the whole file's
geometry follows from that split.

```text
# Phase 1, "parabolic": drag acts, modelled as a LINEAR decay of the initial
# velocity rather than a drag proportional to current speed.
velocity(t) = start_velocity * max(0, 1 - air_resistance * t) + gravity * t
position(t) = start_position + start_velocity * t
              - start_velocity * air_resistance * t^2 / 2
              + gravity * t^2 / 2

# Phase 2, "free fall": once the linear decay would take the horizontal
# velocity to zero, drag stops and only gravity acts.
fall_time     = max(0, 1 / air_resistance - epsilon)
position(t)   = position(fall_time) + velocity(fall_time) * (t - fall_time)
                + gravity * (t - fall_time)^2 / 2
```

**Invariants** — the crossover for *position* is at one over the drag coefficient; the
crossover for *velocity* is at **two** over it. The two are genuinely different times and
using one for the other breaks the continuity of the curve.

The small epsilon subtracted from the crossover keeps the horizontal velocity from being
exactly zero there, which several downstream divisions require.

**Notes** — linear velocity decay is not physical drag. It is a model chosen because it has
a closed form, which the sweep needs. It produces a bullet that loses forward speed at a
constant rate and then falls — visually close enough, and exactly reproducible.

A bullet fired straight up or straight down has no horizontal velocity, and the model's
divisions by horizontal speed are undefined. Those cases are detected and fall back to plain
gravity, which the comments describe as "fake" — honest about it being a patch rather than
the model.

In single player the drag coefficient is a **global** from configuration and the cartridge's
own value is ignored; in multiplayer the cartridge's is used. Nothing explains the
divergence.

## `process_bullet` — the sweep

**Contract** — advances one bullet across one frame's worth of time, resolving every
collision it meets along the way. Returns whether the bullet is still alive. This is the
routine the whole file exists for.

The problem it solves: the trajectory is *curved*, and the collision database only answers
straight-ray queries. The solution is to approximate the curve with straight segments whose
length is chosen adaptively — long where the curve is nearly straight, short where it bends —
so that the sagitta of each chord stays under a fixed tolerance.

```text
FUNCTION process_bullet(bullet, delta_time) -> alive
  gravity = (0, -gravity_const, 0)
  drag    = global drag in single player, else the cartridge's
  bullet.tracer_start_position = bullet.bullet_pos
  low  = bullet.life_time
  high = bullet.life_time + delta_time
  bullet.change_trajectory_count = 0

  LOOP
    IF bullet.speed < 1              -> RETURN dead   # too slow to matter
    IF bullet.change_trajectory_count >= 32 -> RETURN dead

    # pick how far ahead this straight segment may reach
    time = select_pick_time(bullet, low, high, gravity, drag)
    IF time == low                   -> RETURN dead   # no progress possible

    IF a collision was found in [low, time]
      # the bullet ricocheted or penetrated: a NEW segment starts at the
      # impact, the clock rewinds to zero, and the remaining time shrinks
      # by exactly the time consumed
      high = high - time_consumed
      IF low and high have converged  -> RETURN bullet.speed is non-zero
      CONTINUE                        # re-sweep the remainder
    ELSE
      IF NOT advance the bullet to `time` -> RETURN dead
      IF time reached high            -> RETURN alive
      low = time
```

**Invariants** — the 32-segment cap per frame is a hard bound on how many ricochets and
penetrations one bullet may resolve in one frame, and it exists because the loop can
otherwise fail to terminate on degenerate geometry. It is a safety valve, not a game rule —
but it is observable: a bullet in a very tight enclosure stops existing rather than bouncing
forever.

The minimum speed of 1 unit and the separately configured minimum bullet speed are two
different thresholds and only the hard-coded one is consulted here.

## `trajectory_select_pick_time` — choosing a segment length

**Contract** — chooses the far end of the next straight segment, by binary search, under
two bounds: the segment must not exceed the bullet's remaining range, and the curve's
deviation from the chord must stay under a tenth of a unit.

```text
FUNCTION select_pick_time(bullet, low, high) -> time
  # bound 1: the bullet's remaining range. Which formula applies depends on
  # which phase of the trajectory the interval lies in.
  IF the whole interval is in free fall      -> bound by gravity-only range
  ELSE IF the whole interval is parabolic    -> bound by binary search on arc length
  ELSE                                       -> split at the phase crossover
  IF the range bound already ends the interval, take it

  # bound 2: the chord error. The worst deviation of the curve from the chord
  # is at the MIDPOINT in time, which holds for this trajectory family.
  binary search `time` downward until:
    error(low, time) < 0.1
  where error(a, b) = the perpendicular distance from the position at the
                      midpoint time to the chord joining the positions at a and b
  RETURN time
```

**Invariants** — that the maximum deviation occurs at the midpoint *in time* is asserted in
a comment and is true for a trajectory that is quadratic in time, which both phases are. A
rebuild changing the ballistic model must re-derive it rather than assume it.

**Notes** — the tolerance of a tenth of a unit is the sagitta budget: it is what decides how
many collision queries a long shot costs. Tightening it makes distant shots more accurate
and more expensive; there is no evidence a sweep of values was tried.

The arc length used for the range bound is measured as two chords — start to midpoint,
midpoint to end — rather than integrated, which under-measures a curve and so over-estimates
range slightly.

## `trajectory_check_error` — one segment against the world

**Contract** — issues one straight ray query along the chord and, if anything is hit, splits
the bullet's trajectory there. Returns whether the segment completed *without* a collision.

```text
FUNCTION check_segment(bullet, low, high) -> no_collision
  start  = position(low); target = position(high)
  chord  = target - start; distance = |chord|
  IF distance is zero -> RETURN no_collision
  bullet.dir = normalized chord
  bullet.flags.ricochet_was = false
  ray query from start along chord for `distance`, against both static and
    dynamic geometry, with a per-candidate filter and a per-result handler

  IF nothing hit, or the handler reported no collision time
    RETURN no_collision

  # a collision: begin a NEW trajectory segment at the impact point
  high = high - collision_time            # what is left of the frame
  low  = 0
  change_trajectory_count += 1
  bullet.start_position  = impact point
  bullet.bullet_pos      = impact point
  bullet.start_velocity  = bullet.dir * bullet.speed   # both set by the handler
  bullet.born_time      += collision_time in milliseconds
  bullet.life_time       = 0
  RETURN collided
```

**Invariants** — the direction and speed after impact are decided by the hit handler, which
may reflect the bullet (ricochet), reduce its speed (penetration), or zero it (stop). The
sweep does not know which happened; it only knows a new segment starts.

## `firetrace_callback` — what a ray hit means

**Contract** — invoked for the *nearest* hit along a segment. Converts the ray's hit range
back into a time along the curve, updates the bullet's speed and direction at that time, and
registers a deferred hit event carrying the material that was struck. Always stops the trace
after the first accepted hit.

```text
FUNCTION on_ray_hit(result, data) -> continue_tracing
  data.collide_position = bullet.bullet_pos + bullet.dir * result.range
  update the bullet's speed and direction at the corresponding TIME on the
    curve (see below) — not at the ray's parameter
  IF the bullet's speed is now zero -> stop
  IF the collision time is zero     -> continue tracing past this hit
  IF the hit was static geometry
    material = the struck triangle's material
  ELSE
    IF the object has no skeleton -> stop
    material = the struck bone's material
  register a deferred hit event
  stop
```

**Invariants** — the ray reports a *distance along the chord*, but the bullet's state must be
evaluated at a *time along the curve*. The two conversions (`update_bullet_parabolic` and
`update_bullet_gravitation`) invert the position formula for the appropriate phase to recover
that time. Skipping the conversion and using the chord parameter directly would make fast
bullets land at the wrong speed.

The material comes from the *bone* on a dynamic target, not from the object. That is what
makes a headshot different from a leg shot before any damage model runs: the skeleton carries
material identity per bone.

## `CommitEvents` — applying the frame's hits

**Contract** — runs at the **start** of a frame, on the main thread, and is the only place a
bullet may change the world. Walks the deferred event list, applying each hit and performing
each removal, then clears the list.

```text
FUNCTION commit_events()
  IF more than 1000 events, warn         # a diagnostic only, nothing is dropped
  FOR EACH event
    hit:    apply to a dynamic or a static target
    remove: report it to the multiplayer statistics if it was tracked,
            then remove by swapping the last bullet into its slot
  clear the event list
```

**Invariants** — removal by swap-with-last is what forces the sweep to iterate in reverse
and to carry the index in the event. A rebuild using stable handles or tombstones is free of
both constraints.

## `RegisterEvent`

**Contract** — records a deferred hit or removal, copying the *whole bullet* into the event.
The copy is essential: the bullet continues, ricochets, and may be removed before the event
is applied, so the event must carry the state as of the impact.

For a hit it also computes the transferred power and impulse immediately, and decides whether
this is a *repeated* hit on the same target. The repeat rule differs by mode:

- Single player: every hit updates the remembered target, so a bullet passing through two
  limbs of the same creature registers the second as a repeat.
- Multiplayer: the remembered target is updated only when the struck bone does **not** pass
  bullets through — so a bullet passing cleanly through a non-solid bone does not count as
  having hit that target at all.

**Notes** — the repeat flag exists to let the damage model discount subsequent hits from one
bullet on one body, which is what stops a bullet crossing a torso from applying full damage
twice.

## `CommitRenderSet`

**Contract** — runs at the **end** of a frame. Copies the entire working set into a separate
rendering set, then either schedules the sweep on a worker thread or runs it inline.

**Invariants** — the copy is what makes the parallel sweep safe: the renderer reads the
copy, the sweep mutates the original, and the two never meet. It is a full copy of every
live bullet every frame, which is the price of the parallelism.

## `UpdateWorkload` — the sweep driver

**Contract** — advances every bullet by the frame's elapsed time scaled by a configurable
factor, in **reverse order**, registering a removal for each one the sweep reports dead. A
zero elapsed time is a no-op.

**Notes** — the time scale factor is a global, configurable multiplier on bullet velocity in
time. It exists so a level or a mod can slow projectiles without changing any cartridge.

## `AddBullet` / `SBullet` construction

**Contract** — creates a bullet from a shot: a position, a direction, a speed, a damage pair,
the firer and weapon identities, a maximum range, and a cartridge that scales nearly all of
it. Must be called from the main thread.

```text
FUNCTION construct_bullet(position, direction, speed, power, impulse, ..., cartridge)
  position and velocity from the arguments; speed must be positive
  power   = power   * cartridge.hit_factor
  impulse = impulse * cartridge.impulse_factor
  max_dist = max_dist * cartridge.distance_factor
  armour piercing, drag, wallmark size, tracer colour, bullet material
    all come from the cartridge
  flags for tracer, ricochet, explosive and magnetic beam come from the
    cartridge's own flags
  init_frame_num = the current frame        # it may not draw until the next
```

**Notes** — the cartridge is the scaling layer over the weapon: the weapon supplies the base
numbers and the loaded round multiplies them. That is how armour-piercing and tracer rounds
are expressed without per-weapon tables.

A "one tracer in five" rule is implemented immediately above a line that then overwrites the
tracer flag unconditionally from the cartridge. **The five-round rule is dead code** — every
round of a tracer cartridge draws a tracer. That is a defect, and a visible one.

## `Render`

**Contract** — draws a short streak behind each bullet that has one, as camera-facing
geometry in one batch. Its subtleties are all about making a fast-moving point legible:

```text
FOR EACH bullet IN the rendered copy
  SKIP unless it allows a tracer, tracers are enabled, and it may draw this frame
  streak = current position - position at start of frame
  SKIP IF shorter than the minimum length; CLAMP to the maximum
  # a tracer passing near the camera is widened, so a round going past your
  # head reads as motion rather than as a dot
  width = base width, scaled down when the streak is far from the camera
  # a tracer must not be drawn through the camera
  IF the camera is closer than the streak is long
    shorten it to just short of the camera
  is_own = the firer is the entity the player sees through
  emit the streak
draw the whole batch with culling off
```

**Invariants** — culling is disabled for the batch and restored afterwards, because the
streaks are two-sided quads oriented toward the camera.

**Notes** — the near-camera widening uses two squared distances, one and nine hundredths, as
its range. Neither is explained. The three-tenths-of-a-unit pull-back from the camera is
similarly a magic clearance.

Whether the firer is the viewer is passed down to the tracer renderer, so your own tracers
can be drawn differently from other people's — the first-person view sees them from behind
rather than from the side.

## `Load`

**Contract** — reads the whole tuning set from configuration, from a *different section* in
multiplayer when one is declared. Gravity, drag, the minimum speed, the two collision-energy
bounds, the hit-probability range and the three tracer dimensions all come from there, as do
the ricochet whine sounds and the explosion particle effects.

**Notes** — a separate multiplayer section for ballistics is a first-class feature, not a
patch: the two modes are expected to want different ballistics, and a rebuild should keep the
indirection.

## `PlayWhineSound` / `PlayExplodePS`

**Contract** — the two presentation side effects. A ricochet whine plays from the struck
object's position, chosen at random from the configured set, and only for a firearm hit, and
only once per bullet — the handle on the bullet is the guard. An explosive round spawns one
of the configured particle effects at the impact, queued on the persistent game rather than
owned here, so it outlives the bullet.

## `Clear`

**Contract** — drops every bullet and every pending event, without applying either. Used on a
level change: bullets do not survive one.

## `CalculateNewVelocity`

**Contract** — declared as a step of the velocity model and unused; the phase-aware velocity
function replaced it.
