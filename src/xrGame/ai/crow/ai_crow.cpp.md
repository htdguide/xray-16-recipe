# src/xrGame/ai/crow/ai_crow.cpp

> The crow: ambient flying decoration that circles the player, can be shot down, and becomes a lootable corpse when it lands.

**Needs** — [`ai_crow.h`](ai_crow.h.md) · [`Level.h`](../../Level.h.md) · [`Actor.h`](../../Actor.h.md) · [`Hit.h`](../../Hit.h.md) · [`script_game_object.h`](../../script_game_object.h.md) · [`game_object_space.h`](../../game_object_space.h.md) · [`xrPhysics/PhysicsShell.h`](../../../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../../../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`ai_crow.h`](ai_crow.h.md)
**Tier floor** — T2: a per-frame flight integrator and a ragdoll handoff; the arithmetic is small and the layout is nobody's business but its own

## Purpose

The crow is the cheapest creature in the game and the clearest demonstration of what the
rest of chapter 24 is paying for. It has no perception, no memory, no enemies, no
navigation mesh position, no planner and no state machinery beyond a single identifier.
It flies toward a wandering goal point that is re-rolled near the player every few seconds,
caws occasionally, and when hit at all — regardless of damage — it dies, grows a ragdoll,
falls, and on landing re-enters the AI-visible set so that creatures and scripts can find
the corpse.

It is a separate file, and not part of the monster hierarchy, precisely so that a level can
hold dozens of them without any of them costing what a real creature costs.

## State

```text
RECORD Crow
  current_state   : {flying_level, climbing, falling_dead, landed_dead, undefined}
  target_state    : same                # the selector writes here; the tick applies it
  goal            : vec3                # a point in the air it is steering toward
  orientation     : (yaw, pitch, roll)  # integrated, not derived from a physics body
  previous_pos    : vec3                # last tick's position; the climb test compares heights
  yaw_rate        : real                # smoothed turn rate, carried between ticks

  # tuned from the configuration section
  speed           : real                # world units per second, forward along facing
  angular_speed   : real
  goal_period     : real                # seconds between goal re-rolls
  min_height      : real                # how far above the player the goal is placed
  goal_spread     : vec3                # box half-extents the goal is jittered within
  call_period     : real                # seconds between idle calls

  goal_timer      : real                # counts down; re-roll at or below zero
  call_timer      : real
  play_death_idle : bool                # set by the animation-finished callback

  workload_frame  : int                 # last frame the flight integrator ran
  workload_rframe : int                 # last frame the crow was drawn
```

**Invariants** — `current_state` is what the animation currently shows; `target_state` is
what the selector wants. The two are reconciled exactly once per scheduled update, and the
entry action for a state runs only on that reconciliation, never twice. Once the state
reaches *landed dead* it never leaves.

## Configuration

Every tuned number above is read from the crow's configuration section by name: `speed`,
`angular_speed`, `goal_change_delta`, `min_height`, `goal_variability`,
`idle_sound_delta`. The constructor seeds them with defaults, and the section overwrites
all of them, so the defaults matter only if a section is incomplete. The call timer is
seeded with a random offset of up to half a period so that a flock spawned in one frame
does not caw in unison.

## `load`

**Contract** — reads the tuned parameters from the section, creates the idle-call sound
family, and — the load-bearing part — **removes itself from the AI-visible and
sound-reactive spatial sets**. A living crow is invisible to every other creature's
perception and deaf to every sound event. That is a performance decision with a visible
consequence: stalkers do not shoot at crows, and a crow does not scatter when a gun goes
off. Only the landed corpse re-enters the AI-visible set.

## `spawn`

**Contract** — becomes visible and enabled, resolves the five animation groups from the
model, and branches on health. A crow spawned alive starts in *flying level* with its
per-frame update **switched off** — it runs only on the scheduler's coarse update until
something hits it. A crow spawned with no health (a corpse restored from a save, or an
authored dead crow) starts in *falling dead*, switches its per-frame update on and builds
its ragdoll immediately.

**Notes** — the animation groups are resolved against two candidate base names each
(`death`/`norm_death`, `fly_fwd`/`norm_fly_fwd`, and so on) because two generations of crow
model ship with different naming. Each group must resolve to at least one motion or the
spawn fails outright; there is no animation-less crow.

## `scheduled_update`

**Contract** — the crow's whole brain, run at the scheduler's rate rather than every frame.
Applies any pending state change, runs the selector, re-rolls the goal and the call on
their timers, and moves the sound sources to the current position. Allocates nothing.

```text
FUNCTION scheduled_update(dt_ms)
  dt = dt_ms / 1000
  spatial.remove(visible_to_ai)          # re-asserted every tick; only the landed
                                         # corpse puts it back, and it does so after this

  IF target_state != current_state THEN
    enter(target_state)                  # exactly one entry action per transition
    current_state = target_state

  # selector: two conditions, both comparing this tick's height to last tick's
  IF current_state = flying_level AND position.y > previous_pos.y THEN
    target_state = climbing
  ELSE IF current_state = climbing AND position.y <= previous_pos.y THEN
    target_state = flying_level
  ELSE IF current_state = falling_dead THEN
    run_falling_dead()

  IF current_state is neither falling_dead nor landed_dead THEN
    IF goal_timer <= 0 THEN
      goal_timer = goal_period jittered by ±50%
      goal = actor.position lifted by min_height, jittered within goal_spread
    goal_timer = goal_timer - dt

    IF call_timer <= 0 THEN
      call_timer = call_period jittered by ±50%
      play one random idle call at this position
    call_timer = call_timer - dt

  move all sound sources to position

  IF not drawn in the last two frames THEN run_flight(dt)
```

**Invariants** — the climb/level distinction is derived *only* from whether the crow gained
height since the previous tick; there is no climbing intent anywhere. The two states exist
to select between two flight animations, and nothing else reads them.

**Notes** — the goal is always placed relative to the *player*, not relative to the crow.
Crows therefore circle wherever the player is, which is the entire point of the creature:
they are set dressing that follows the camera. A rebuild that anchors the goal to the
crow's own position gets crows that wander off and are never seen again.

The ±50% jitter on both timers is applied to the period each time it is re-armed, not once,
so a flock stays desynchronized rather than drifting back into phase.

## `run_flight`

**Contract** — one step of the flight integrator. Steers the crow's orientation toward the
goal and advances its position along its own facing. Deterministic given the state; no
allocation; does not consult the collision database, so a crow will fly through geometry.

```text
FUNCTION run_flight(dt)
  turn = angular_speed * dt
  offset = goal - position

  # pitch: a dead band, then a clamped ramp, then damping inside the band
  IF offset.y > 1 THEN       pitch = min(pitch + turn,  0.8)
  ELSE IF offset.y < -1 THEN pitch = max(pitch - turn, -0.8)
  ELSE                       pitch = pitch * 0.95

  # yaw: flatten both vectors, steer by how far off we are, sign by the cross product
  flatten offset and facing into the horizontal plane, normalize both
  misalignment = (1 - dot(facing, offset)) / 2 * turn * 10
  sign = sign of cross(offset, facing).y
  yaw_rate = (yaw_rate * 9 + sign * misalignment) / 10     # heavy smoothing
  yaw  = yaw + yaw_rate
  roll = -yaw_rate * 9                                     # bank into the turn

  previous_pos = position
  orientation = from(yaw, pitch, roll)
  position = previous_pos + facing * speed * dt
```

**Invariants** — the pitch clamp of ±0.8 radians (about 46 degrees) is what stops the crow
flipping over when the goal is directly above or below it. The dead band of one world unit
in height is what stops it oscillating around a goal at its own altitude.

**Notes** — the roll is *nine times the yaw rate* and the yaw rate is smoothed with a
nine-to-one filter. Neither number is derived anywhere; together they are the crow's entire
flight feel. The `(1 - dot)/2` term maps alignment onto zero-to-one so that a crow facing
away turns hardest, which is the standard shape and needs no explanation, but the factor of
ten that follows it does not have one.

The integrator is driven from **two** places: from the render path, with the elapsed time
scaled by however many frames were skipped, and from the scheduled update when the crow has
not been drawn for more than two frames. A frame guard makes sure it runs at most once per
frame whichever path reached it first. This is the crow's answer to a problem every ambient
creature has — it must keep moving when off-screen, but it must move *smoothly* when
on-screen, and the scheduler's coarse tick is too lumpy for that. A rebuild with a uniform
update rate does not need the dual path at all.

## `run_falling_dead`

**Contract** — watches the ragdoll's vertical velocity and declares the fall over once the
body stops descending. With no ragdoll (the physics shell failed or was never built), the
fall is over immediately. Also consumes the one-shot flag the animation layer sets when the
death animation finishes, starting the in-air death idle.

**Notes** — the downward acceleration vector assembled at the top of the routine is never
used; gravity comes from the physics world. It is dead code.

## `die`

**Contract** — switches the per-frame update on, builds the ragdoll, and fires the scripted
death callback with the killer. The ordering is load-bearing and marked as such in the
original: the per-frame update must be enabled **before** the ragdoll is built, because
building it enables processing again and the paired disable would underflow.

## `hit_signal`

**Contract** — any hit kills a crow. Health is set to zero regardless of damage, and unless
the crow has already landed the target state becomes *falling dead*. A hit on an
already-landed corpse just replays the landed animation.

**Notes** — the crow is the only creature in the game with no damage model. This is not an
oversight: a crow exists to be shot at, and the interesting outcome is the falling body,
not a wounded bird.

## `hit`

**Contract** — forwards the hit to the base entity with its **impulse divided by 100**, then
fires the scripted hit callback. The division is what keeps a rifle round from launching the
ragdoll out of the level; the base entity's impulse scale is tuned for human-sized targets.

## `create_physics_shell`

**Contract** — deliberately does nothing, overriding the base entity's automatic shell
creation. The crow's shell is built only at death, from `die` or from a dead spawn, because
a living crow is animated by the flight integrator and must not be driven by the solver.

## `per_frame_update`

**Contract** — when a ragdoll exists, steps it and copies its transform onto the crow. That
is the only thing the per-frame path does, which is why it is switched off entirely while
the crow is alive.

## `render`

**Contract** — runs the flight integrator for however many frames have elapsed since it last
ran, then draws, then records the frame. See the dual-drive note under `run_flight`.

## `network export` / `network import`

**Contract** — the wire form is health, the server time, a flags byte that is always zero,
the position, four angle values, and the team/squad/group triple.

**Notes** — the angle block writes the yaw **twice** and then the pitch and a literal zero;
the reader reads four values into yaw, yaw, pitch and roll, so the first write is
overwritten on receipt and the roll is always zero. A remote crow therefore never banks.
The duplicated field is load-bearing only in that the two ends agree on the field count; a
rebuild defining its own protocol should send three angles or one.

## `animation group` / `sound group`

**Contract** — each loads a family of at most eight variants by appending `_0`, `_1`, …
to a base name, falling back to a second base name per variant, and requires at least one to
resolve. Sampling is uniform over whatever loaded. The sound family additionally checks that
the audio file exists before creating a source, because creating a source for a missing
file is fatal.

**Notes** — the cap of eight is a fixed-capacity inline array, chosen so that the groups sit
inside the crow rather than in separate allocations. No shipped crow has that many variants.
