# src/xrGame/PHMovementControl.cpp

> The bridge between "I want to walk there" and a physically simulated body: it turns a desired path or an input acceleration into forces on a character body, and reports back what the body hit on the way.

**Needs** — [`PHMovementControl.h`](PHMovementControl.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`CaptureBoneCallback.h`](CaptureBoneCallback.h.md) · [`Level.h`](Level.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`xrPhysics/PHCharacter.h`](../xrPhysics/PHCharacter.h.md) · [`xrPhysics/IPHCapture.h`](../xrPhysics/IPHCapture.h.md) · [`xrPhysics/ElevatorState.h`](../xrPhysics/ElevatorState.h.md) · [`xrPhysics/IColisiondamageInfo.h`](../xrPhysics/IColisiondamageInfo.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: per-physics-step force control against a solver, with hard step-rate coupling

## Purpose

Every creature in the game — the player, a stalker, a dog — moves as a **box sliding on the
ground under force control**, not as a kinematically placed transform. This object is the
layer between what the creature's mind decided and what the solver is told.

It answers two very different callers with the same body underneath:

- **The player**, who supplies an acceleration and a camera direction every frame, plus jumps.
- **An AI**, which supplies a *path* and a speed, and expects this object to work out which
  way to push.

The second is the substantial half. The path is a polyline of navigation points; the body is
somewhere near it but never on it; and the direction to push is the tangent of the path,
corrected toward the path so that drift is pulled out. All the path geometry in this file is
in service of that one output.

It also owns everything else about a character body that the game layer needs: its collision
boxes and the switching between them (standing, crouching, prone), collision damage, the
"am I on the ground, at a wall, or in the air" classification, jump trajectory solving, the
ability to grab a physical object, and the injurious-volume counter.

## State

```text
RECORD PHMovementControl
  character         : PHCharacter     # the body in the solver; may not exist
  character_type    : { actor, ai }
  parent_object     : GameObject
  capture           : optional<Capture>  # a held physical object
  # placement, mirrored
  position          : position        # this layer's copy, kept in step with the body
  velocity          : vector
  actual_velocity   : real            # the smoothed magnitude the animation system reads
  # collision boxes
  boxes             : 4 boxes         # stand, crouch, ... one per posture
  aabb              : box             # the active one
  current_box       : int
  trying_times      : 4 timestamps    # last failed attempt to switch to each box
  trying_poses      : 4 positions
  # path following
  path_dir          : direction       # the output: which way to push
  path_point        : position        # the nearest point on the path
  path_distance     : real            # how far off the path the body is
  path_size         : int
  start_index       : int             # where to resume the nearest-point search
  exact_position    : bool            # next update should snap to the body, not search
  # environment and damage
  environment, old_environment : { on_ground, at_wall, in_air }
  mass, min_crash_speed, max_crash_speed, collision_damage_factor : real
  contact_speed     : real
  gcontact_was      : bool            # landed this step
  gcontact_power    : real            # impact as a fraction of the lethal speed
  gcontact_health_lost : real         # the resulting damage, 0..1
  block_damage_step_end : int         # physics step until which collision damage is ignored
  in_dead_area_count: int             # signed count of injurious volumes entered
  material          : int
  external_impulse  : vector          # accumulated pushes from outside
  air_control_param : real
  non_interactive   : bool            # present but collides with nothing
```

**Invariants**
- `position` is this layer's mirror of the body's position, refreshed at the start of every
  update and written back on every explicit placement. Every path computation uses the mirror,
  never the solver's copy, so the solver may be stepping concurrently.
- `exact_position` means "the body was just teleported; do not search the path for where we
  were, take where we are". It is set by placement and cleared by the first update after.
- The box array is fixed at four. Switching between them is not free — see the dynamic
  activation below — and a failed switch is remembered per box with a timestamp and a position
  so it is not retried until the body has moved or half a second has passed.
- `in_dead_area_count` is a **signed** count incremented on crossing into an injurious volume
  and decremented on crossing out, judged by which way the crossed surface faced. It is
  positive when inside. Miss a crossing and it never recovers.

## `Calculate` (player form)

**Contract** — one frame of player-driven movement. Takes the desired acceleration, the
camera direction, a jump flag, and pushes them into the body; then reads back the resulting
velocity, the landing impact and the environment.

```text
FUNCTION calculate(accel, cam_dir, ang_speed, jump, dt, light)
  previous = position
  position = the body's interpolated position
  IF an external impulse is pending
    accel = accel + impulse; push the impulse into the body as a force; clear it

  body.cam_dir = cam_dir                     # the body leans and steps relative to view
  body.max_velocity = |accel| / 10           # the SPEED LIMIT comes from the acceleration
  body.acceleration = accel
  IF jump is non-zero
    body.jump(accel)

  velocity = body's saved velocity; actual_velocity = |velocity|
  landed = body reported a ground contact
  update collision damage
  IF a collision-damage callback is installed
    call it with the crash speed bounds, the contact speed and the damage
  trace the segment previous -> position for injurious-volume crossings
  classify the environment
  body.reinit()
```

**Invariants** — the maximum velocity is derived from the *magnitude of the requested
acceleration*, divided by ten. That is the load-bearing oddity of player movement in this
engine: the caller expresses "how hard to push" and the top speed follows from it, so a
sprint and a walk differ in both at once and cannot be varied independently. The divisor of
ten has no stated derivation.

The body is *reinitialized* at the end of every player update and not after an AI one. That
asymmetry is not explained.

## `Calculate` (path-following form)

**Contract** — one frame of AI-driven movement. Given a path, a speed and the index of the
travel point last reached, works out which way to push and how fast, and advances the travel
point.

```text
FUNCTION calculate(path, speed, travel_point, precision)
  IF this is a leaping monster with an enemy, replace the path with a two-point
    path to the enemy and compute a lateral correction (see the jump aim below)

  IF non-interactive
    position = the object's own position; write it into the body; RETURN
  IF the body does not exist -> RETURN

  new_position = the body's interpolated position

  IF the path is empty
    speed = 0; position = new_position
  ELSE IF exact_position                      # just teleported: do not search
    direction = toward the next path point, flattened; take it as-is
    path_distance = 0; path_point = position
  ELSE IF the path has one point
    speed = 0; aim straight at it
  ELSE
    # find the nearest point on the polyline, searching OUTWARD from where we
    # were last time rather than from the start. The search radius is twice
    # how far the body moved since the last known path point — so a body that
    # barely moved searches barely at all.
    radius = 2 * |new_position - path_point|
    search up from start_index, then down from start_index, both within radius
    IF nothing was found within radius
      search the WHOLE path from the beginning       # the fallback
    IF the nearest feature is a segment
      direction = path_dir_line(index, precision)
    ELSE
      direction = path_dir_point(index, precision)
    travel_point = index; start_index = index
    IF speed is zero, direction = zero

  flatten direction to horizontal; normalize

  IF an external impulse is pending
    combine it with (direction * speed * 10) and re-derive both
    # the factor of ten converts between this layer's "speed" and a force scale
    push the impulse into the body; clear it

  body.max_velocity = speed
  body.acceleration = direction (or the jump deviation, when aiming a leap)
  velocity = the body's SMOOTHED velocity; actual_velocity = |velocity|
  landed = body reported a ground contact
  update collision damage
  classify the environment
  exact_position = false
```

**Invariants** — the incremental nearest-point search is what makes path following affordable
for a level full of creatures: a path can be hundreds of points long and is searched over a
window proportional to how far the body moved. The whole-path fallback exists because the
window search fails whenever the body is thrown, teleported or pushed hard.

The player form reads the body's *saved* velocity and the AI form reads its *smoothed*
velocity. The AI's animation blending needs a stable number; the player's camera needs a
responsive one.

**Notes** — a monster mid-leap with an enemy has its path replaced entirely by a two-point
path to that enemy, and its acceleration replaced by a purely **lateral** correction — the
component of "toward the enemy" perpendicular to its current velocity, scaled by eight and by
how far into the jump it is. That is the auto-aim on a leaping creature: it cannot change its
speed mid-air, only steer sideways, and it steers harder the longer it has been airborne.
The factor of eight is unexplained.

## `PathNearestPoint` and its two windowed forms

**Contract** — find the point on the polyline closest to a position, reporting the distance,
the point, the segment tangent, the index, and whether the nearest feature was a *segment* or
a *vertex*. The three forms differ only in what part of the path they scan: the whole thing,
upward from a start index until the distance exceeds a radius, or downward likewise.

```text
FOR EACH segment (first, second) of the path
  IF the position is BEFORE this segment
    IF it was also AFTER the previous one
      # the two exclusions leave exactly the vertex between them
      candidate = first; near_line = false
    mark "not after a line"
  ELSE IF the position is BEFORE the second point
    # inside the segment's slab: project onto it
    candidate = first + tangent * (offset along tangent); near_line = true
  ELSE
    mark "after this line"
IF nothing matched at all
  # the position is past the end of the whole path
  candidate = the last point; near_line = false
```

**Invariants** — the segment/vertex distinction is not cosmetic. It selects which of the two
direction routines runs, and those behave differently: on a segment the body is steered
along the tangent and nudged toward the line; at a vertex it is steered *around* the corner
on a tangent circle. Getting it wrong makes a creature cut corners or overshoot them.

The "before this segment and after the previous one" pair of exclusions is how a vertex is
detected without testing vertices: the region belonging to neither adjoining segment's slab
is exactly the wedge outside the corner.

## `PathDIrLine` / `PathDIrPoint` / `CorrectPathDir`

**Contract** — turn the nearest-point result into a push direction. The common shape: take the
path's own direction, then add a correction toward the path scaled by a caller-supplied
*precision*.

```text
FUNCTION path_dir_line(index, precision) -> direction
  along = corrected path direction at this index
  toward = (path_point - position), normalized
  # the correction saturates: beyond one foot radius off the path it is a
  # constant pull, within it the pull tapers to zero at the path. Without the
  # taper the body oscillates across the line.
  IF distance > foot_radius THEN toward = toward * precision
  ELSE                           toward = toward * distance * precision
  RETURN normalize(along + toward)

FUNCTION path_dir_point(index, precision) -> direction
  # rounding a corner: steer along the TANGENT of a circle about the corner,
  # on whichever side the body is already turning, plus the same saturating
  # pull toward the corner.
  IF this is the last point of the path -> steer straight at it
  tangent = up cross (direction to the corner), flipped to match current heading
  RETURN normalize(tangent + the same saturating pull)
```

**Invariants** — `CorrectPathDir` handles a degenerate tangent: when the path's own direction
is almost vertical (a ladder, a drop) it has no horizontal component to steer by, so the
routine **looks ahead to the next segment** — recursively — for a usable horizontal direction.
Without it a creature on a vertical path has no heading at all.

**Notes** — `path_dir_point` computes a normalization before testing whether the magnitude is
near zero, so a body exactly on a corner divides by a near-zero length before the guard that
would have caught it. The guard below then discards the result, so it is harmless in
practice, but the order is wrong.

Both routines take a `distance` argument and neither uses it; they read the stored path
distance instead.

## `UpdateCollisionDamage`

**Contract** — converts the body's reported contact speed into damage, once per update.

```text
FUNCTION update_collision_damage()
  contact_speed = the body's reported contact velocity
  IF collision damage is blocked and the block has not expired
    contact_speed = 0; RETURN               # e.g. just after a teleport
  impact_power = contact_speed / max_crash_speed
  IF contact_speed > min_crash_speed
    # damage ramps linearly from nothing at the minimum to full at the maximum
    health_lost = (contact_speed - min_crash_speed)
                  / (max_crash_speed - min_crash_speed)
    the hit type is chosen from the material struck
```

**Invariants** — the two crash speeds are per-creature configuration and define a **ramp, not
a threshold**: below the minimum a fall is harmless, above the maximum it is fully lethal,
and between them damage is linear in speed. That ramp is the entire fall-damage model.

The damage block is measured in *physics steps*, not in time, because it is set immediately
after an operation that moves the body discontinuously and must survive exactly as long as the
solver needs to settle.

**Notes** — the hit type is derived from the material underfoot: an injurious material in
single player turns a fall into radiation damage rather than impact damage, and the two older
games use a distinct "physical strike" type in multiplayer. That is three different answers
to "what kind of damage is a fall" across the shipped titles.

The collision damage factor from configuration is **squared** before being handed to the body.
Nothing explains the squaring; it means the configured value is not the multiplier it appears
to be.

## `TraceBorder` / `BorderTraceCallback`

**Contract** — after every move, casts a ray along the segment just travelled against *static*
geometry only, and adjusts the injurious-volume counter by which way each crossed injurious
surface faced.

```text
FUNCTION on_border_hit(triangle)
  IF it is not static geometry -> ignore, keep tracing
  IF the triangle's material is injurious
    IF we crossed it moving AGAINST its normal -> entered:  count += 1
    ELSE                                       -> left:     count -= 1
  keep tracing        # every crossing in the segment matters, not just the first
```

**Invariants** — the trace continues through every hit rather than stopping at the first,
because a single frame's movement can enter and leave several volumes. The facing test is what
makes entry and exit distinguishable from one-sided geometry.

**Notes** — the counter is a running signed total with no resynchronization. A teleport, a
level change, or a movement fast enough to skip a boundary leaves it permanently wrong, and
"am I in an injurious volume" is then wrong forever. A rebuild should recompute it from
containment periodically.

## `ActivateBox` / `ActivateBoxDynamic` / `InterpolateBox` / `SetBox`

**Contract** — posture switching. `ActivateBox` changes the collision box outright.
`InterpolateBox` blends partway toward another box, which is how the crouch transition is
animated rather than snapped. `ActivateBoxDynamic` is the careful form.

The careful form exists because **growing a collision box can put the body inside geometry**.
Standing up under a low ceiling must fail rather than eject the character through the floor.

```text
FUNCTION activate_box_dynamic(id, iterations, steps, resolve_depth) -> succeeded
  # do not retry a switch that failed recently from the same place
  IF this box failed within the last 500 ms AND the body has moved under 0.05
    RETURN false
  IF the object has a full physics shell instead of a character body
    RETURN false
  IF the box is already active -> RETURN true
  IF no body exists yet, create one temporarily

  save velocity and position
  attempt the resize, letting the solver push the body out of penetration over
    `iterations` iterations and `steps` steps, accepting up to `resolve_depth`

  IF it failed
    undo everything: destroy or re-disable the temporary body, restore the old
    box, restore velocity, straighten the body's rotation, restore position
    remember the failure: this box, at this position, at this time
  ELSE
    commit the new box; clear the failure record
  restore velocity either way
  RETURN whether it succeeded
```

**Invariants** — the failure memory (a timestamp and a position per box) bounds the cost of
repeatedly trying to stand under an overhang: without it, a creature holding the stand key
under a ledge would run the full penetration resolve every frame. Half a second, or five
centimetres of movement, is what it takes to earn another attempt.

The body's rotation is explicitly straightened on failure. The resolve can leave a character
capsule tipped, and a tipped character body is a bug that never recovers on its own.

## `VirtualMoveTo`

**Contract** — answers "where would this body end up if it walked to there", **without
actually moving it**. Used to validate a destination before committing to it.

```text
FUNCTION virtual_move_to(target) -> reachable position
  save the body's complete state, its contact callback and its gravity setting
  install a no-op collision callback; disable initial-contact handling;
    turn gravity OFF                       # a probe must not fall
  compute the number of steps, capped at 20, to cover the distance at unit speed
  derive the force that covers exactly that distance in that many steps
  FOR EACH step
    zero the velocity; apply the force; step the body alone
  read the resulting position
  restore everything that was saved
```

**Invariants** — the velocity is zeroed *before every step*, so each step is an independent
push rather than an accumulation. That makes the probe travel at a controlled rate regardless
of what the body was doing, and is what makes the result depend only on geometry.

The save-and-restore is exhaustive — state, callback, gravity, contact handling — and it must
be, because the probe steps the real body in the real world. A rebuild with a scratch body
avoids the whole dance.

**Notes** — the twenty-step cap means a probe cannot reach further than twenty steps at unit
speed. Beyond that the answer is silently truncated rather than reported as unreachable.

## Jumping

**Contract** — four entry points at increasing levels of abstraction: jump with a given
velocity; jump from here to there in a given time; jump to there in the minimum-speed time;
and ask what the minimum-speed time would be.

```text
FUNCTION jump(start, end, time)
  velocity = the throw velocity that carries start to end in `time` under gravity
  enable the body; apply it

FUNCTION jump(end) -> time
  time = the time of the minimum-initial-speed trajectory to `end`
  jump(here, end, time); RETURN time
```

The trajectory is additionally **classified** into one of three shapes, which is what the
animation layer needs:

- *straight* — the landing point is reached before the apex; the arc only rises.
- *high* — the landing point is the apex.
- *curved* — the landing point is past the apex; the arc rises and falls.

**Invariants** — the classification compares the time to the apex (vertical speed over
gravity) against the total flight time. It is what lets a creature play a rising leap versus
an arcing pounce from the same code.

**Notes** — the classification routine computes its velocity from `position - end` while the
jump itself uses `end - position`: the vector is **inverted** relative to the actual jump.
The subsequent test also reads the caller's *output* parameter before it is written. Both
look like defects, and whether the classification ever returns the right answer is not
recoverable from the source.

## `Load`

**Contract** — reads the creature's physical parameters from its configuration section: two
collision boxes (a centre and a half-size each, so box 0 and box 1 are the two postures), the
two crash speeds, the mass, an optional collision-damage factor and an optional restrictor
class naming which navigation restrictions this creature obeys.

**Invariants** — the collision damage factor must not exceed one: a creature may be made
*less* fragile than the ramp says, never more.

**Notes** — only two of the four boxes are read from configuration. The other two are set by
whoever needs them, which means the meaning of box 2 and box 3 is not declared anywhere.

The friction constants for ground, air and wall are defined at the top of the file and every
use of them is commented out. Friction now lives entirely in the body.

## `ApplyHit`

**Contract** — how being hit affects movement. Only the player is affected. When standing on
ground or against a wall, a hit of a *physical* kind — burn, shock, strike, wound — zeroes the
velocity outright: you are stopped in your tracks. The non-contact kinds — radiation, psychic,
chemical, light burn — do not interrupt movement. Explosion and heavy wound hits additionally
apply a physical impulse, so they push as well as stop.

**Invariants** — stopping is instantaneous velocity zeroing, not a force. That is why being
shot while sprinting feels like hitting a wall rather than like being pushed.

**Notes** — the switch is written with every case falling into the next and only two cases
actually acting, which makes the commented intent ("stop" / "not stop" per case) the only
readable statement of the rule. The table above is that intent; the code as written stops on
burn, shock, strike and wound, and does nothing for anything else.

## Object capture

**Contract** — a character may grab a physically simulated object and carry it. `PHCaptureObject`
begins a capture, either at a bone chosen by a callback or at a named element, refusing if a
capture is already in progress or the target has no active body. `PHReleaseObject` ends it.
The capture is polled each scheduled update and destroyed when it reports failure — the held
object was destroyed, moved too far, or could not be held.

**Notes** — the helpers that find the nearest element of a target to the character, and its
transform, exist so the grab point is chosen by proximity rather than authored.

## `SetNonInteractive`

**Contract** — puts the body in a mode where it exists and is positioned but collides with
nothing and is disabled in the solver. Used when something else — an animation, a vehicle, a
cutscene — owns the character's placement.

**Invariants** — in this mode the path-following update copies the object's own position into
the body rather than the other way round. The direction of authority reverses.

## Environment classification

**Contract** — `CheckEnvironment` asks the body whether it is on the ground, against a wall or
in the air, and keeps both the current and the previous answer. The previous answer is what
makes a *landing* detectable as a transition rather than a state.

`SetEnvironment` allows both to be forced, which a save restore uses.

## Freeze / UnFreeze / enable / disable / velocity / position accessors

**Contract** — the plumbing to the body: freeze and unfreeze (used by correction prediction),
enable and disable, get and set velocity and position, read smoothed velocity, read ground
normal, read the last material stood on and the injurious material touched, read foot radius,
enable and disable collision, and install contact callbacks for the body as a whole and for
its feet separately.

**Invariants** — a foot contact callback separate from the body's is what makes footstep
sounds possible: the sound is chosen by the material under the *foot*, at the moment of
contact, which is not the same as the material the body is resting on.

Every accessor tolerates a non-existent body and answers a neutral value — zero velocity, an
upward ground normal. A character can exist in the game without a body in the solver, which is
what an offline or a non-interactive creature is.

## `CreateCharacter` / `AllocateCharacterObject` / `DestroyCharacter` / `DeleteCharacterObject`

**Contract** — the body's lifecycle, in two stages: allocate the right *kind* of body (the
player's and an AI's differ, and the player's differs again between single player and
multiplayer), then create it in the solver from the active box's dimensions, its material, its
air-control factor and its collision damage factor.

**Invariants** — allocation and creation are separate because the kind is chosen once and the
body may be created, destroyed and re-created many times — every posture probe does exactly
that when no body exists yet.

## `UpdateObjectBox`

**Contract** — computes how wide this character appears to *another* character, projected onto
the line between them and weighted by the camera direction, and hands that as an effective
radius so the other one's navigation restriction can account for it. Only applies within two
metres.

**Notes** — this is how two characters avoid each other in a doorway: each tells the other how
much room it needs from that particular angle, rather than both using a fixed radius. The
camera weighting means the effect is strongest for characters in front of the player, which is
a presentation compromise inside what looks like a simulation routine.

It calls the player-form `Calculate` with zero acceleration purely to refresh the mirrored
position, which is an expensive way to read a position.

## `SetPathDir`

**Contract** — sets the steering direction, with a range check that logs and asserts on a
component above a thousand. It is a trap for a corrupted direction propagating from a
degenerate path, and the assertion is the only thing standing between that and a creature
accelerating to infinity.

## `BlockDamageSet` / `NetRelcase`

**Contract** — suppress collision damage for a number of physics steps (after a teleport or a
forced placement), and drop any reference to an object the engine is destroying — in
particular a captured object, which would otherwise be held after its death.
