# src/xrPhysics/PHWorld.cpp

> The fixed-timestep loop: accumulate real elapsed time, run whole steps, and inside each step run collision, tune, solve, and read-back in a strictly fixed order.

**Needs** — [`PHWorld.h`](PHWorld.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PHCommander.h`](PHCommander.h.md) · [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`params.h`](params.h.md) · [`console_vars.h`](console_vars.h.md) · [`GeometryBits.h`](GeometryBits.h.md) · [`ExtendedGeom.h`](ExtendedGeom.h.md) · [`dRayMotions.h`](dRayMotions.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`PHWorld.h`](PHWorld.h.md)
**Tier floor** — T1: hands the dynamics library its global tuning and owns the level mesh shape handle.

## Purpose

This file answers one question: *what happens, in what order, when physics advances*. Everything
else in the module is machinery serving that order. The ordering is not incidental — the source
marks one line "the order is critical" and several invariants elsewhere depend on phases not being
reshuffled.

## State

```text
RECORD PhysicsWorld
  objects                  : ItemList<PhysicsObject>   # stepped every step
  frozen_objects           : ItemList<PhysicsObject>   # held still, not stepped
  recently_disabled        : ItemList<PhysicsObject>   # asleep, still holding cached collision data
  update_objects           : ItemList<UpdateObject>    # non-collidable step participants
  frozen_update_objects    : ItemList<UpdateObject>

  level_mesh               : shape handle              # the whole static triangle soup, as one shape
  motion_ray               : shape handle              # one shared swept-motion shape, reused per query
  spatial_results          : list<spatial entry>       # one shared query buffer, reused per object

  frame_time               : real       # leftover time not yet consumed by a whole step; in [0, fixed_step)
  previous_frame_time      : real       # the same, one frame ago
  frame_parity             : bool       # flips whenever a step actually ran
  steps_taken              : int (64-bit)
  gravity                  : real
  disable_countdown        : int
  world_frozen             : bool
  processing               : bool       # true only inside a step; several operations assert on it

  commander                : DeferredCallList          # condition/action pairs, see PHCommander
  statistics               : {collision time, solver time, total}

  # invariant: frame_time < fixed_step at the end of every frame
  # invariant: an object is in exactly one of {objects, frozen_objects}; recently_disabled is
  #            an overlay on neither — a recently-disabled object is in no stepped list
  # invariant: processing is false whenever an object is created, destroyed, frozen or thawed
```

## `create` / `destroy`

**Contract** — `create` loads tuning from configuration, registers the world as a per-frame
consumer, creates the shared contact constraint group, the level-mesh shape and the shared motion
ray, publishes gravity and the global error-reduction and constraint-force-mixing parameters to the
dynamics library, seeds the world boundary box from the level's bounding volume, and initializes
the collision-group filter. `destroy` reverses it and shuts the library down.

**Invariants** — the world boundary is the level's bounding volume **lowered by 30 metres at the
bottom**. Anything below that floor is considered escaped and is disabled rather than simulated
(see `out_of_boundaries` in [`Physics.h`](Physics.h.md)). The margin exists because a body can
legitimately be a little below the authored level geometry — inside a pit, or mid-fall through a
hole — and the check must not fire on those. Thirty metres is a judgement call with no derivation.

## `on_frame`

**Contract** — the engine's per-frame entry point. Times the whole physics pass, and hands the
frame's elapsed real time to the accumulator.

## `frame_step` — the accumulator

**Contract** — converts elapsed real time into a whole number of fixed steps, runs them, and
carries the remainder. Does nothing while the world is frozen.

```text
FUNCTION frame_step(elapsed)
  IF world_frozen THEN RETURN
  REQUIRE elapsed IS FINITE
  elapsed := elapsed * time_factor         # a script-settable global slow-motion dial

  pending := frame_time + elapsed
  IF pending < fixed_step THEN
    frame_time := pending                  # not enough for a step; nothing simulates this frame
    RETURN

  steps       := floor(pending / fixed_step)
  previous_frame_time := frame_time
  frame_time  := pending - steps * fixed_step
  frame_parity := NOT frame_parity         # tells interpolation which pair of samples is current

  processing := true
  FOR i FROM 0 TO steps - 1: step()
  processing := false
```

**Notes** — the leftover `frame_time` is not waste; it is the *interpolation phase*. Rendering
happens between steps, and [`PHInterpolation.cpp`](PHInterpolation.cpp.md) blends the last two
solved placements by `frame_time / fixed_step` to produce the transform the renderer draws. This is
why the physics rate and the display rate are decoupled without visible stutter.

There is **no upper bound on `steps`** here — only a diagnostic message past twenty. That
contradicts the runtime invariant in
[§6 of the system requirements](../../SYSTEM-REQUIREMENTS.md#6-conformance) ("the fixed-timestep
accumulator never advances more than a fixed number of steps in one frame"), which is enforced
elsewhere in the engine, not here. A rebuild should cap the burst at this level: a long hitch
otherwise produces a catch-up burst that causes the next hitch.

## `step` — the fixed-step order

**Contract** — advances every active object by exactly one fixed step. This ordering is the
module's central contract.

```text
FUNCTION step()
  # 0. age the sleep bookkeeping, once every N steps rather than every step
  IF disable_countdown = 0 THEN
    disable_countdown := configured_sleep_check_period
    FOR EACH obj IN recently_disabled: obj.check_recently_deactivated()
  IF NOT world_frozen THEN disable_countdown := disable_countdown - 1
  steps_taken := steps_taken + 1

  # 1. COLLISION — every object generates its contacts; islands merge as pairs are found
  FOR EACH obj IN objects: obj.collide()

  # 2. TUNE — pre-solve. Contacts exist; bodies may still be adjusted.
  FOR EACH obj IN objects:        obj.tune(fixed_step)
  FOR EACH obj IN update_objects: obj.tune(fixed_step)

  # 3. deferred script calls whose condition has come true
  commander.run_due_calls()

  # 4. SOLVE — one solve per live island
  FOR EACH obj IN objects: obj.island.step(fixed_step)

  # 5. READ BACK — unmerge, then let each object digest its own result and re-index itself
  FOR EACH obj IN objects
    obj.island.unmerge()
    obj.data_update(fixed_step)
    obj.recompute_bounds_and_reindex()
  FOR EACH obj IN update_objects: obj.data_update(fixed_step)

  # 6. release this step's contacts — MUST be after read-back
  release all contact constraints, their force feedback records and contact effectors

  # 7. report the simulated time window to whoever asked (the bullet manager)
  IF step_time_callback EXISTS THEN callback(window_start, window_start + fixed_step)
```

**Invariants** — step 6 after step 5 is explicitly marked critical in the source, and the reason is
that breakables read *joint force feedback* during read-back (see
[`PHFracture.cpp`](PHFracture.cpp.md)): the forces a contact applied are only readable while the
contact constraint still exists. Free them first and every breakable silently stops breaking.

Step 5's unmerge-before-read-back matters for the same family of reasons: `data_update` may move or
disable a body, and both are only legal on an unmerged island.

**Notes** — the collision phase and the solve phase are timed separately and reported, because they
are the two halves that scale differently: collision with object *count and proximity*, solving
with island *size*. A rebuild should keep both counters; the performance criterion in
[§6](../../SYSTEM-REQUIREMENTS.md#6-conformance) is unlikely to be met without knowing which half
is the problem.

## `step_touch`

**Contract** — a single collision-and-wake pass with **no solve**. Collides everything, wakes every
body, unmerges, re-indexes, and drops the contacts. Used to answer "is this configuration
penetrating" without advancing time — the character-restriction resolver and shell deactivation
both use it.

## `freeze` / `unfreeze`

**Contract** — move *all* objects between the active and frozen lists in constant time, then call
the content hook on each. Asserts on double-freeze. The pair exists so one object can be stepped in
isolation (`freeze` the world, `unfreeze` the one object, `step`) without the rest of the level
drifting.

## `calc_num_steps`

**Contract** — how many whole steps a given elapsed-millisecond interval will produce, given the
accumulator's current remainder. Used by the deferred-call layer to express "in N milliseconds" as
"at step number X" (see [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md)).

## `set_step`

**Contract** — changes the fixed step and re-derives the global spring and damping constants so
that *the material response stays the same physical stiffness* at the new rate.

```text
FUNCTION set_step(s)
  fixed_step := s
  # the base pair (cfm, erp) was authored at base_fixed_step; convert it to
  # spring/damping (rate-independent), then convert back at the new rate
  spring  := spring_from(base_cfm, base_erp, base_fixed_step)
  damping := damping_from(base_cfm, base_erp)
  world_cfm := cfm_from(spring, damping, fixed_step)
  world_erp := erp_from(spring, damping, fixed_step)
  IF the world exists THEN re-seed the accumulator remainder for the new step
```

**Notes** — this conversion is the reason [`PhysicsCommon.h`](PhysicsCommon.h.md) exists. The
dynamics library's soft-constraint parameters are *rate-dependent*; the authored material data is
not. Every place that sets a contact's softness goes through the same conversion. Changing the step
without re-deriving makes every surface in the game harder or softer.

## `get_state`

**Contract** — snapshots every element of every active object into a flat list of
(synchronizable, state) pairs. This is the multiplayer determinism check: both ends step and
compare.

## `net_relcase`

**Contract** — a shell is about to be destroyed by a network correction. Drops every deferred call
that references it, and gives every update object the chance to drop its own reference.

## `add_call`

**Contract** — registers a condition/action pair with the deferred-call list, thread-safely. This
is the script-facing hook: Lua can ask for something to happen at a future simulation step. The
pair is checked in phase 3 of every step, which means script-scheduled effects land inside the
simulation rather than between frames — necessary for them to be deterministic.

## `create_physics_world` / `destroy_physics_world` / `physics_world`

**Contract** — the module's construction seam. `physics_world` returns the abstract interface from
[`IPHWorld.h`](IPHWorld.h.md), which is what `xrEngine` and `xrGame` hold; the concrete type never
crosses the module boundary.

## `CPHMesh`

**Contract** — creates and destroys the one shape handle standing for the level's static triangle
soup. Creation registers it with the shared geometry-bit table so the collider knows which class it
belongs to.
