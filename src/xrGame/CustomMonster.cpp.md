# src/xrGame/CustomMonster.cpp

> The base every thinking creature is built on: it owns the senses, the memory, the movement and the sound player, splits the frame between a slow thinking tick and a fast presentation tick, and interpolates the creature's visible pose from a queue of past states.

**Needs** — [`CustomMonster.h`](CustomMonster.h.md) · [`CustomMonster_inline.h`](CustomMonster_inline.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`script_entity.h`](script_entity.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`movement_manager.h`](movement_manager.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`sound_player.h`](sound_player.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`moving_object.h`](moving_object.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame vector and angle work, a bounded state queue, and worker-thread dispatch

## Purpose

Every creature with a brain — the stalkers, every mutant — is this class plus a brain. It
is the junction where six independent subsystems are wired together and given a frame
budget, and the wiring is the substance:

- **the senses** ([*feel*](../../GLOSSARY.md)) — vision, sound and touch, each an opt-in
  interface the creature implements;
- **memory** — what the creature has seen, heard, been hurt by and wants, and how it ranks
  those things;
- **movement** — pathfinding, the detail path, and the physical character controller;
- **the sound player** — the creature's own voice;
- **the script entity** — the hook by which a script can take a creature over entirely;
- **the network state queue** — the interpolated pose.

**Thinking is slow, presentation is fast.** The creature thinks on the scheduler's tick,
which is between 100 and 250 milliseconds; it is drawn and posed every frame. The bridge
between the two rates is a queue of timestamped states: the thinking tick *appends* a
state, the frame tick *interpolates between two of them* at a fixed lag behind the server
clock. This machinery is the multiplayer client's interpolation, reused unchanged in
single player, where the creature is producing the very states it is interpolating. That
is why a single-player creature's visible position lags its authoritative position by the
configured latency — and why the code carefully preserves the authoritative position when
it owns the creature.

**Vision is expensive, so it is split across frames and threads.** A vision update is two
halves — build the frustum and query what falls inside it, then raycast against the
candidates — and they run on alternate ticks. Either half may be dispatched to a worker
thread. Nothing else in the creature is threaded.

## State

```text
RECORD NetUpdate                  # one timestamped snapshot of the visible pose
  timestamp   : int               # server game time
  model_yaw   : real              # the body's facing
  torso       : rotation          # yaw, pitch, roll of the upper body, in world terms
  position    : vector
  health      : real

RECORD CustomMonster
  memory, movement, sound_player  : owned subsystems, created before the base constructs
  sound_visitor                   : owned; interprets an emitted sound's AI attributes
  moving_object                   : owned; the creature's entry in the spatial index

  eye_bone        : int           # which skeleton bone the eyes sit on
  eye_matrix      : transform     # the eye's world frame, rebuilt each vision tick
  eye_fov         : real          # degrees
  eye_range       : real          # metres
  eye_shift       : vector        # per-species offset from the bone to the actual eye
  eye_shift_yaw   : real
  eye_stage       : int           # alternates the two halves of the vision update

  states          : queue<NetUpdate>
  last_state      : NetUpdate     # the interpolated pose actually applied this frame
  was_interpolating, was_extrapolating : bool

  panic_threshold           : real   # health fraction below which a creature panics
  far_plane_factor          : real   # how much weather shortens this creature's sight
  fog_density_factor        : real
  killer_clsids             : list<class id>   # who is allowed to kill it specially
  invulnerable              : bool

  # critical wounds
  critical_wound_threshold        : real  # negative disables the whole mechanism
  critical_wound_decrease_quant   : real  # accumulator bleed per second
  critical_wound_accumulator      : real
  critical_wound_type             : optional<int>
  last_hit_time                   : int
  bones_body_parts                : map<bone, wound type>
```

Invariants that are not written down anywhere else:

- the state queue is **never empty** after spawn — spawn seeds it with two entries, and the
  trim keeps at least two. Both the export path and the frame tick assert on it.
- the trim removes a state only when the *next* one is already older than the interpolation
  time, so the pair bracketing the current render time always survives.
- the per-frame elapsed time is clamped to 100 milliseconds. A long hitch therefore
  advances the sound player and everything keyed to it by at most that, rather than by the
  real gap.
- the critical wound accumulator is clamped into `[0, threshold]` — it can never bank more
  than one wound's worth of damage.

## `_construct` / `~CCustomMonster`

**Contract** — the subsystems are created *before* the base class constructs, because the
base's construction already reaches into them. Each is created through an overridable
factory, so a species can substitute its own memory or movement manager. Destruction
releases all four, tells the level's sound system to forget this listener, and clears the
creature's entry in the deferred-spawn registry.

**Invariants** — the "tell the level to forget me" step is the conformance invariant about
dangling references, and it is needed here because sound delivery holds listeners by
pointer across frames.

## `Load` / `reload` / `reinit`

**Contract** — three separate configuration passes, and the split is real:

- `Load` is the one-time read: the surface material table, the physical movement
  parameters, the memory and movement subsystems' own sections, and the eye's field of
  view and range. It also nudges the creature up by an epsilon — see Notes.
- `reload` re-reads what may change when a creature's section changes underneath it: the
  sound bank, the material table, movement, the special-killer class list, the two weather
  sensitivity factors and the panic threshold.
- `reinit` resets runtime state to a freshly-spawned condition: the interpolation clocks,
  the vision stage, the eye offset, and the entire critical-wound accumulator. It also
  reads the two critical-wound numbers and, if the mechanism is enabled, asks the species
  to load its bone-to-wound-type table.

**Notes** — the epsilon lift at load is a spawn-placement guard: an authored position
exactly on the floor plane can fail the physical character controller's ground test. It is
the kind of fix a rebuild will need its own version of, not the same number.

## `net_Export` / `net_Import`

**Contract** — the wire form of a creature is one state plus its team affiliations. The
export sends the *most recent queued state*, not the current position, so that what the
server publishes and what the owner is interpolating agree. The import appends the received
state to the queue, but only if it is newer than the queue's tail — out-of-order packets
are dropped rather than reordered.

```text
FUNCTION export() -> packet
  REQUIRE we own this creature AND the queue is non-empty
  latest = queue.back
  write health, latest.timestamp, a reserved byte,
        latest.position, latest.model_yaw,
        latest.torso.yaw, latest.torso.pitch, latest.torso.roll,
        team, squad, group

FUNCTION import(packet)
  REQUIRE we do not own this creature
  read health and apply it
  read a state and the three affiliations
  IF the queue is empty OR the state is newer than the tail THEN
    append it; mark that we are interpolating
  make the creature visible and enabled
```

**Notes** — the angles are sent as full-precision reals. The original's own comments show
they were once quantized to a byte each, and a rebuild designing its own protocol should
quantize: a creature's facing does not need 32 bits.

## `shedule_Update` — the thinking tick

**Contract** — the rate-degraded update, and the order within it is load-bearing.

```text
FUNCTION scheduled_update(dt_ms)
  # 1. trim the state queue to what interpolation still needs
  render_time = server_time - interpolation_latency
  WHILE queue.size > 2 AND queue[1].timestamp < render_time
    drop queue.front

  # 2. senses and memory, but only while alive
  IF alive THEN
    run one half of the vision update, on a worker thread if configured
    memory.update(dt)

  base.scheduled_update(dt_ms)

  IF dt > 3 seconds THEN RETURN          # a hitch: skip a think entirely

  # 3. think, unless something else is driving this creature
  IF we do not own this creature THEN
    (nothing: a remote creature is posed by its imported states alone)
  ELSE
    IF a script has taken control THEN run the script
    ELSE IF enough frames have passed since spawn THEN think()

    # 4. act on what thinking decided, then publish the result
    IF health > 0 THEN act(dt)
    append a new state: now, the body's current yaw and torso rotation,
                        the current position, the current health
```

**Invariants**

- the state is appended on **both** branches of the health test — a dead creature still
  publishes states, because its corpse still has to be interpolated into position.
- the think is suppressed for a configured number of frames after spawn, so that a level
  full of freshly spawned creatures does not all think on the same frame.
- skipping the think when the tick was longer than three seconds prevents a creature from
  integrating a huge delta after a load or a hitch.

**Notes** — the "we do not own this creature" branch is empty and the comment beside the
position update calls the whole networking arrangement fake. In single player the creature
is always locally owned, so the remote path is barely exercised.

## `UpdateCL` — the presentation tick

**Contract** — the per-frame update. Advances the sound player, chooses between
interpolation and extrapolation, computes the visible pose, and writes it into the
creature's transform.

```text
FUNCTION frame_update()
  frame_delta = min(now - last_frame_time, 100 ms)     # clamped; see State
  base.frame_update()
  deliver any pending animation sound callbacks to the script layer
  advance the sound player, on a worker thread if configured

  IF the queue is empty THEN update the animation controller; RETURN

  render_time = server_time - interpolation_latency
  IF render_time is past the newest state OR fewer than two states exist THEN
    # extrapolation: nothing newer exists, so hold the newest
    last_state = queue.back
  ELSE
    find the adjacent pair bracketing render_time
    factor = how far render_time lies between them
    last_state = interpolate(pair, factor)             # angles lerped as angles
    IF we own this creature THEN
      restore last_state.position to the pre-interpolation position   # see Notes
    ELSE
      pick an animation from the interpolated facing, path direction and speed

  IF we own this creature AND it is alive THEN
    drive the physical movement toward the state's position,
    and pick an animation from the result

  IF alive AND no animation is driving the transform THEN
    set the transform's yaw from the model yaw
    set the transform's origin from the position
    post-multiply the torso pitch onto it

  update the animation controller
```

**Invariants**

- **an owned creature's position is never interpolated.** The interpolation runs, and then
  the position component is thrown away and replaced with what it was. Only the *angles*
  are interpolated for an owned creature. This is what keeps the authoritative position
  authoritative while still smoothing the facing between thinking ticks.
- the transform is only written when no animation controller owns it. An animation that
  moves the creature — a leap, a takedown — takes the transform over completely, and
  anything that writes it behind the controller's back corrupts the animation. The
  original checks this in three separate places and asserts it at four more in debug.
- angles are interpolated with an angle-aware lerp that takes the short way round, not a
  linear blend; a creature turning through the wrap point would otherwise spin the long
  way.

## `Exec_Visibility` and the three eye stages

**Contract** — the vision update, split across two ticks. Stage zero rebuilds the eye frame
and issues the frustum query; stage one raycasts the candidates. They alternate, so a
creature does a full vision cycle every two thinking ticks.

```text
FUNCTION visibility_tick()
  IF dead THEN RETURN
  IF stage is even THEN build the eye frame; query the frustum
  ELSE                   raycast the candidates
  stage = stage + 1
```

`eye_pp_s0` — **build the eye frame.** Forces the skeleton to pose, takes the eye bone's
world transform for its *position*, and builds the frame's *orientation* from the
creature's head rotation rather than from the bone. The eye is then offset by a
per-species shift.

**Notes** — taking position from the bone and orientation from the head-rotation state is
deliberate: the bone's orientation follows the animation, which lags and overshoots, while
the head rotation is what the creature has decided to look at. A creature must see where
it intends to look, not where its neck currently is.

`eye_pp_s1` — **query the frustum.** Builds a camera-space view matrix from the eye frame
and a projection from the current field of view and range, then asks the senses system
which objects fall inside. The range is first modulated by the weather.

`eye_pp_s2` — **raycast.** Advances the vision system's per-object visibility accumulators
by the elapsed time, testing line of sight against the collision database and against the
material transparency of anything in the way. Visibility is therefore *gradual*: an object
becomes seen after being in the frustum and unoccluded for long enough.

## `update_range_fov`

**Contract** — modulates the creature's sight range by the weather. The engine's current far
plane and fog density are read from the environment, and the range is scaled down by both:
by how far the far plane has closed in relative to the creature's nominal range, and
inversely by the fog density. Field of view is untouched.

```text
FUNCTION weather_adjusted_range(base_range) -> real
  far   = environment.far_plane        # 300 in clear weather, 50 in heavy fog
  fog   = environment.fog_density      # 0 none, 1 full
  reach = min(far_plane_factor * far, eye_range) / eye_range
  RETURN base_range * reach * (1 / (1 + fog_density_factor * fog))
```

**Invariants** — the two factors are per-species, so a mutant that hunts by smell can be
given a far-plane factor that makes weather irrelevant to it while a human stalker is
blinded by fog. This is the mechanism by which the game's weather has gameplay
consequences and not merely visual ones.

## `net_Spawn`

**Contract** — brings the creature up. The order is strictly load-bearing.

```text
FUNCTION spawn(record)
  memory.reload(section); memory.reinit()      # BEFORE anything can consult memory
  movement.spawn(record)                       # must succeed
  base.spawn(record)
  script_entity.spawn(record)

  declare this object visible to AI, and — only if alive — reactive to sound

  eye frame = identity
  set both current and target body yaw from the record's torso yaw, negated
  set pitch to zero
  set health from the record
  IF not alive THEN stamp the death time

  IF this creature uses navigation positions AND it has no parent THEN
    adopt the record's game-graph vertex
    IF the record's next game vertex is on this level and reachable THEN
      set it as the movement destination
    IF the creature's own navigation vertex is reachable THEN
      set it as the level destination
    ELSE
      find the nearest reachable vertex and go there instead

  eye_bone = the bone named by the section's "bone_head" key

  IF we own this creature THEN
    seed the state queue with TWO identical states, one at
    (now - latency) and one at now
    make it visible and enabled

  scheduler rate = between 100 and 250 milliseconds
  create the spatial-index entry
```

**Invariants**

- seeding *two* states is what makes the very first frame interpolate rather than
  extrapolate; a single seed would leave the creature snapping.
- the unreachable-spawn fallback matters: level data places creatures at positions that
  their own movement restrictors forbid, and without the fallback such a creature would
  never move. The recovery is to path to the nearest position it *is* allowed to occupy.
- the yaw from the record is negated. The authored and the runtime angle conventions differ
  in handedness; this negation appears at every boundary between them.

## `net_Destroy`

**Contract** — tears down in the reverse order: base, script entity, sound bank, movement.
Then — and this is the part that matters — **removes both worker-thread dispatch entries**
for this creature, destroys the spatial-index entry, and clears its visibility contribution.
A creature destroyed while one of its parallel tasks is queued would otherwise have that
task run against freed memory.

## `feel_touch_contact` / `feel_touch_on_contact` — the anomaly special case

**Contract** — the touch sense's admission tests, overridden for exactly one purpose: a
creature is in contact with an anomaly only if the creature's *origin point* is inside the
anomaly's volume, not if their bounding volumes merely overlap. Everything else is admitted
unconditionally.

**Notes** — without this, a creature brushing past a large anomaly's bounding sphere would
register as inside it. The two variants differ only in the radius of the test sphere
(epsilon against zero), which is a distinction without a difference.

## `feel_visible_isRelevant`

**Contract** — the vision system's pre-filter: a creature only considers living things on
other teams. Everything else is excluded before any raycast happens, which is the single
biggest saving in the vision system.

## `feel_sound_new`

**Contract** — an emitted sound reaching this creature. Dead or destroyed creatures drop it;
otherwise it is handed to the sound memory, which interprets the emitter's AI attributes to
decide what the creature now believes.

## `Hit` / `HitSignal` / `Die`

**Contract** — damage passes through to the base unless the creature is flagged invulnerable,
in which case it is dropped entirely — not reduced, not recorded. The hit-reaction signal is
empty in the base; species override it. Death clears the creature's contribution to the
player's visibility, so a corpse does not keep the player visible to it.

## `update_critical_wounded`

**Contract** — the mechanism behind a creature losing the use of a limb rather than simply
dying. Damage to a bone accumulates into a single scalar which bleeds away over time; when
it crosses a threshold, and the species' own conditions permit, the bone is looked up in a
bone-to-wound-type table and the corresponding critical wound starts.

```text
FUNCTION update_critical_wound(bone, power) -> bool
  REQUIRE not already critically wounded
  IF threshold < 0 THEN RETURN false           # the species has the mechanism disabled

  elapsed = seconds since the last hit, or 0 on the first
  accumulator = accumulator + power - decrease_quant * elapsed
  clamp accumulator into [0, threshold]
  last_hit_time = now

  IF accumulator < threshold THEN RETURN false

  reset the accumulator and the hit clock
  IF the species' external conditions are not suitable THEN RETURN false
  IF bone is not in the body-part table THEN RETURN false
  wound_type = table[bone]
  start the critical wounded state
  RETURN true
```

**Invariants** — the accumulator resets the moment it *crosses* the threshold, whether or
not a wound actually results. So a creature whose conditions are wrong, or whose hit bone
has no mapping, loses the banked damage and must accumulate again. That makes critical
wounds much rarer than the threshold alone suggests, and it is load-bearing for the feel:
a limb is lost to a run of hits on the same place, not to a long tally of scattered ones.

**Notes** — the decrease is applied as a rate against the *time since the last hit*, so a
creature hit steadily accumulates almost nothing away and a creature hit once every few
seconds never accumulates at all.

## The evaluation delegations

**Contract** — six methods, in three pairs — is this item / enemy / danger worth caring
about, and how much — each forwarding to the corresponding memory sub-manager. They exist
as virtual methods on the creature so that a species can override its priorities without
replacing a manager.

**Notes** — the managers call *back* into the creature to ask these questions, so a species
overriding one changes how its own memory ranks things. That inversion is the extension
point.

## `PitchCorrection`

**Contract** — aligns the creature's body pitch to the slope of the navigation cell it
stands on, so that a creature on a ramp leans with it. The cell's three-vertex contour
defines a plane; the creature's facing is projected onto that plane and the resulting
direction's pitch becomes the body's target.

**Notes** — the pitch is negated on assignment, the same handedness mismatch as at spawn.

## `create_anim_mov_ctrl` / `destroy_anim_mov_ctrl`

**Contract** — the handoff to and from an animation that moves the creature itself. Taking
control disables the movement manager and remembers whether it was enabled. Releasing
control restores that, and then — critically — **reads the creature's angles back out of
the transform the animation left it in** and writes them into both the body rotation state
and the interpolation's last state.

**Invariants** — without that read-back, the frame after an animation ends would snap the
creature back to the facing it had when the animation began. The negations on both angles
are again the handedness convention.

## `ForceTransform`

**Contract** — teleports the creature, through the physical character controller. In
multiplayer it additionally suppresses damage for two seconds afterwards, because a forced
move can push a body through geometry and the resulting collision damage would be
undeserved.

## `spatial_sector_point`

**Contract** — the point used to decide which visibility sector the creature is in: its
origin raised by half its radius. Raising it matters because a creature's origin is at its
feet, and feet are frequently on the far side of a floor from the body.

## `save` / `load`

**Contract** — persistence delegates to the base and then, **only if the creature is alive**,
to memory. A dead creature's memory is not saved and not restored, so a corpse loads with
no recollection of anything. Both sides test the same condition, so the streams stay
aligned.

## `Orientation` / `head_orientation` / `predict_position` / `target_position`

**Contract** — read-only views the rest of the game asks for: the body's current rotation,
the head's rotation record, where the creature will be after a given time, and where it is
currently heading. All delegate to the movement manager.

## Could not recover

- The multi-threaded vision dispatch is guarded by a condition that is written `false &&`,
  so the worker-thread path is dead and vision always runs inline. The debug branch behind
  it distinguishes stalkers from other creatures for reasons nobody recorded.
- A large block of physical movement configuration — box extents, foot extents, crash
  speeds, mass, three friction coefficients — is commented out in `Load`. Those parameters
  are now read by the character physics support instead, but the commented block is the
  only place that names them as a set.
- A torso-spin bone callback is defined in comments and unused; the torso pitch is applied
  to the whole transform instead, which is why a creature's whole body pitches rather than
  only its upper half.
- `eye_pp_timestamp` is used to derive the vision raycast's elapsed time but is never
  initialized in `reinit`, so the first vision cycle after a respawn integrates against an
  arbitrary interval.
- `NET_WasExtrapolating` is cleared on the interpolation path and never set anywhere.
