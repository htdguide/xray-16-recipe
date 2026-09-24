# src/xrGame/PhysicObject.cpp

> The generic physics prop: a level object whose visual is driven by a rigid-body shell of one of four shapes, optionally playing a startup animation, and — in multiplayer — interpolating toward the server's authoritative pose.

**Needs** — [`PhysicObject.h`](PhysicObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHSkeleton.h`](PHSkeleton.h.md) · [`PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [`Level.h`](Level.h.md) · [`moving_bones_snd_player.h`](moving_bones_snd_player.h.md) · [`animation_script_callback.h`](animation_script_callback.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`DestroyablePhysicsObject.cpp`](DestroyablePhysicsObject.cpp.md)
**Tier floor** — T2: it hands structs to the rigid-body seam and bit-packs a network message, but no manual memory or byte-level file layout is intrinsic to the decisions here.

## Purpose

Most of the movable furniture in a level is this class: crates, barrels, corpses of props,
doors, hanging chains, swinging signs. It is deliberately one class with a *type* field
rather than four classes, because the shape of the physics body is authored per object in
the spawn record and the rest of the lifecycle is identical for all four.

Three concerns meet here and are worth separating in a rebuild even though the original
fuses them: the body construction (four shapes), the animation-driven variant (a door
whose pose comes from a motion, not the solver), and the multiplayer pose interpolation.

## State

```text
ENUM PhysicObjectType = box | fixed_chain | free_chain | skeleton

RECORD PhysicObject
  type                  : PhysicObjectType    # from the spawn record
  mass                  : real                # from the spawn record; default 10
  collision_hit_callback: optional<callback>  # owned; notified when this body is hit
  anim_blend            : optional<Blend>     # the startup motion, kept so scripts can scrub it
  bones_snd_player      : optional<BoneSound> # per-bone motion-driven sound, if the model declares it
  anim_script_callback  : AnimCallback        # fires a script call when the startup motion ends
  just_after_spawn      : bool
  activated             : bool                # registered with the per-frame update for interpolation
  net_update            : optional<NetUpdate>

RECORD NetUpdate
  samples : queue<PoseSample>                 # invariant: at most 2 entries are ever kept
RECORD PoseSample
  timestamp : int (ms, global clock)
  state     : PhysicsNetState                 # position, orientation, force, torque, velocities, enabled
```

**Invariants** — the sample queue holds at most two entries. Interpolation needs exactly a
pair, and keeping more would let the object lag arbitrarily behind the server; keeping one
would make it teleport. The consequence — stated nowhere in the source — is that the client
always renders the object *extrapolating past* the newest sample rather than between two
past ones; see `interpolate_states`.

`activated` and the presence of samples are coupled: the object registers with the
per-frame update when its first sample arrives and unregisters when the pair is consumed.
A physics prop that nobody is updating costs nothing, which is the whole point of a
dedicated flag rather than always-on updating — a level holds hundreds of these.

## `net_Spawn`

**Contract** — builds the live object from its server record. Order is load-bearing
throughout.

```text
FUNCTION net_spawn(record) -> bool
  type = record.type ; mass = record.mass
  clear collision callback and animation blend
  base spawn                       # builds the physics shell via create_physics_shell, below
  create_collision_model()         # must follow, it needs the visual the base spawn loaded
  skeleton_spawn(record)           # the breakable-joint layer reads its own saved state
  make visible and enabled
  IF shell is not breakable AND no script is bound AND the skeleton layer is not removing
      unregister from the scheduled (coarse-rate) update
  bones_snd_player = build from the model's own configuration, if it declares one
  IF bones_snd_player THEN start it
  just_after_spawn = true ; activated = false
  IF the record carried an initial update payload
      replay that payload through net_import                 # see note
  RETURN true
```

**Invariants** — the scheduled-update unregistration is the performance decision that makes
a level of props affordable. A prop only needs a coarse periodic tick when something can
change without the physics solver: a breakable joint that may snap, or a script bound to
it. Everything else is driven either by the solver (which calls back) or by nothing at all.

**Notes** — the initial-update replay exists because a client joining mid-game receives the
spawn record and the current pose as one message; the pose half is written into a scratch
packet and fed back through the ordinary import path rather than duplicating the decode.
A rebuild with a structured spawn message just carries the pose as an optional field.

## `create_collision_model`

**Contract** — chooses the collision proxy the world queries this object with. A model
whose embedded configuration declares `collide/mesh` true gets a *dynamic mesh* proxy —
the actual deforming triangles; everything else gets a *skeleton* proxy — one box per bone,
following the bone transforms. Replaces any existing proxy.

**Notes** — the mesh proxy is far more expensive and exists for objects whose silhouette
matters to a bullet trace (a chain-link fence, a hanging cloth) where a per-bone box would
let shots through or stop them wrongly. The choice lives in the *model's* user data rather
than the entity's configuration section because it is a property of the geometry, and one
model is used by many sections.

## `CreatePhysicsShell` / `CreateBody` / `CreateSkeleton` / `AddElement`

**Contract** — builds the rigid-body shell according to `type`. Idempotent: a shell that
already exists is kept.

```text
FUNCTION create_body(record)
  IF shell exists THEN RETURN
  CASE box:
      shell = simple single-box shell of the given mass, starting frozen if the
              record's "active" flag is clear
  CASE fixed_chain, free_chain:
      shell = empty shell bound to the model's skeleton
      add_element(parent = none, bone = skeleton root)      # recursive, below
      set total mass
  CASE skeleton:
      shell = shell built from the model's own per-bone shapes, with the record's
              named bones pinned; then apply the physics overrides from BOTH the
              spawn record's inline configuration and the model's user data
  shell.transform = object transform
  shell.air_resistance = (linear 0.001, angular 0.02)
  shell.disable_params = auto-sleep thresholds read from the model's user data
```

```text
FUNCTION add_element(parent_element, bone_id)              # chain shapes only
  element = a box element matching the bone's authored bounding box
  IF that box's half-extent is under 0.05 in length
      grow every half-extent by 0.05                       # see note
  element.mass = 10
  bind the bone's transform to be driven by this element
  IF NOT (type == free_chain AND parent_element is none)
      joint = full-control joint between parent and element,
              anchored at the element's origin,
              axes taken from the parent's local x and y,
              every axis limited to plus or minus a quarter turn
      add joint
  FOR EACH child bone: add_element(element, child)
```

**Invariants** — `fixed_chain` and `free_chain` differ in exactly one place: the free chain
omits the joint at the root, so the whole chain hangs loose; the fixed chain joints its
root to the world. Everything else about them is identical, which is why they are one code
path with one condition rather than two.

**Notes** — the 0.05 minimum half-extent is a degeneracy guard: a bone authored with a
zero or near-zero box produces a body with a degenerate inertia tensor that the solver
turns into infinities. Growing rather than skipping keeps the chain's joint topology
matching the bone hierarchy, which the animation binding depends on.

Element mass is a flat 10 per bone regardless of the record's total mass, and the total is
then imposed on the finished shell, which redistributes it. The per-element value is
therefore only a ratio and its absolute size is meaningless — this is worth knowing before
"fixing" it.

## `RunStartupAnim` / `SpawnInitPhysics`

**Contract** — spawn-time physics initialization is exactly two steps in this order: build
the shell, then start the authored startup motion. The motion name comes from the server
record and is required — a model that can animate with no startup animation named is a
data error, not a silent no-op. Playing it is followed by an immediate forced pose
evaluation so the shell and the visual agree before the first frame.

**Invariants** — the pose must be recomputed *after* the motion is started and *before*
anything reads a bone transform, because the shell's elements were placed from the bind
pose and the startup motion may move them somewhere else entirely (a door authored open).

## `run_anim_forward` / `run_anim_back` / `stop_anim` / `anim_time_get` / `anim_time_set`

**Contract** — script control of the startup motion, which is how a door opens and closes.
Forward and back set the blend playing with its speed's sign forced positive or negative;
both request the end-of-motion callback so the script learns when it finished. Stop pauses
in place. Time get and set scrub the blend; setting outside the motion's length is refused
rather than clamped, and setting forces an immediate pose recomputation so the result is
visible on the same frame the script asked for it.

Every one of these is a no-op on a model that is not animated, with a diagnostic.

**Notes** — running the *same* blend backwards, rather than playing a second motion, is why
a door needs only one authored animation. The sign flip is the entire close mechanism.

## `set_door_ignore_dynamics` / `unset_door_ignore_dynamics`

**Contract** — installs (idempotently — it removes before adding) a contact filter on this
object's shell that suppresses collisions with *other dynamic objects*, while keeping them
with the actor and with anything whose shell has traced geometry.

```text
FUNCTION door_ignore(contact) -> collide?
  other = the object on the far side of this contact
  IF other is unknown OR other is the actor THEN collide            # never ignore the player
  IF other has no physics shell THEN do not collide                  # a creature; walk through
  IF other's shell has traced geometry THEN collide                  # see note
  do not collide
```

**Notes** — a powered door that can be blocked by a dropped can is a bug report; a door
that pushes the player through a wall is worse. This filter is the compromise: the door
sweeps through clutter and creatures but still meets the actor. "Traced geometry" marks
shells whose motion is swept rather than discretely stepped — typically thrown or shot
objects — which must still be stopped by a door or they tunnel through it.

## `UpdateCL` / `PHObjectPositionUpdate`

**Contract** — the per-frame pass. Drives an animator-backed shell from its animation,
interpolates in multiplayer, pumps the animation-end script callback, pushes the shell's
pose into the object transform, and advances the bone-motion sound.

```text
FUNCTION update_frame()
  base update
  IF the shell has an animator THEN advance the shell from its animation
  IF NOT single player THEN interpolate()
  pump the animation-end script callback
  position_update()
  IF bone sound is active THEN advance it by the frame delta

FUNCTION position_update()
  IF no shell THEN RETURN
  IF type == box            THEN step the shell, take its transform verbatim
  ELSE IF shell has animator THEN take the shell's interpolated global transform
  ELSE                           take the shell's interpolated global transform in place
```

**Invariants** — the box case reads the shell's *current* transform; every other case reads
an *interpolated* one. A single box is one body whose transform is exactly the object's, so
there is nothing to interpolate and taking the raw value avoids a frame of lag. A
multi-element shell has no single transform, so one is derived from the elements and must
be smoothed or the object jitters at the solver's step rate rather than the frame rate.

## `net_Export` / `net_Export_PH_Params`

**Contract** — writes this object's pose to a network packet. Single player and any object
carried by a parent write a single zero byte and stop: a carried object's pose is its
carrier's problem, and single player has no wire.

The message is a one-byte header packing a flag mask and the shell's element count,
followed by force, torque, position and an orientation quaternion, then angular velocity
and linear velocity — each of the last two *omitted entirely* when its corresponding
"is zero" flag is set. A trailing byte says whether the shell is awake.

```text
FUNCTION net_export(packet)
  IF carried OR single player THEN write byte 0 ; RETURN
  state = shell's sync state, or just the position if there is no sync object
  header.count = number of sync elements                # REQUIRE it fits in 5 bits
  header.mask  = (state.enabled ? enabled_bit : 0)
               | (angular velocity is zero ? angular_null_bit : 0)
               | (linear  velocity is zero ? linear_null_bit  : 0)
  write header as one byte
  write force, torque, position                         # three vectors of 32-bit reals
  IF the quaternion's magnitude is zero
      substitute the identity-adjacent value (0,0,1,0)  # see note
  write the four quaternion components
  IF NOT angular_null: write angular velocity
  IF NOT linear_null:  write linear velocity
  write 1 if the shell is awake else 0
```

**Invariants** — the element count is asserted to fit in five bits, which caps a physics
prop at 31 synchronized elements. That cap is a wire-format fact, frozen against the other
end of the connection, and a rebuild that widens it must version the protocol.

**Notes** — the two "is zero" flags are the only compression in the message and they pay
because a level's props are overwhelmingly asleep: a resting crate sends twenty-four fewer
bytes per update than a moving one.

The degenerate-quaternion substitution writes a value that is *not* the identity rotation.
It is a guard against a shell that has never been stepped, where a zero quaternion would
make the receiver's slerp produce infinities. Which non-zero value is used is arbitrary and
the receiver will be corrected by the next update; it is not a meaningful orientation.

## `net_Import` / `net_Import_PH_Params`

**Contract** — decodes the same message into a new pose sample stamped with the local
global clock, and enqueues it for interpolation. A zero header byte means "nothing sent"
and returns immediately. A locally-owned object decodes and discards — the local simulation
is authoritative for it.

```text
FUNCTION net_import(packet)
  header = read byte ; IF header == 0 THEN RETURN
  sample.timestamp = now (global clock)
  read force, torque, position, quaternion
  sample.state.enabled = header.mask has enabled_bit
  angular velocity = header.mask has angular_null ? zero : read three reals
  linear  velocity = header.mask has linear_null  ? zero : read three reals
  previous position and orientation = the just-read values     # see note
  read and discard the awake byte
  IF this object is locally owned THEN RETURN
  register this object with the level's correction-prediction list
  append sample ; WHILE more than 2 samples: drop the oldest
  IF NOT activated THEN register with the per-frame update; activated = true
```

**Notes** — seeding `previous position/orientation` with the incoming values rather than
the object's current ones means an imported pose has zero implied velocity for whatever
consumes those fields. The shell's own velocity fields carry the motion; the previous-pose
pair is used for collision sweeping, and seeding it with the *old* local pose would sweep a
false path across the teleport the correction introduces.

## `Interpolate` / `interpolate_states`

**Contract** — moves a remotely-owned, visible, un-carried object toward the server's pose
once per frame. With one sample it snaps to it; with two it blends between them by a factor
derived from the clock and writes the result into the shell's sync object. When the factor
passes one the older sample is retired and, if that empties the pair, the object
unregisters from the per-frame update.

```text
FUNCTION interpolate_states(first, last, out) -> real
  IF now == last.timestamp THEN RETURN 0
  factor = (now - last.timestamp) / (last.timestamp - first.timestamp)
  clamp a copy of factor to [0,1] for the blend, but RETURN the unclamped value
  out.position    = lerp(first.position, last.position, clamped factor)
  out.orientation = spherical lerp of the two orientations at the clamped factor
  out.previous_position    = out.position
  out.previous_orientation = out.orientation
```

**Notes** — the numerator is measured from the **newest** sample, not the oldest, so the
factor is at least one on the very first frame after a sample pair forms and the blend is
always pinned at the newer pose. The effect is a snap-to-latest with the interval
arithmetic left over from an interpolator that was never finished — the unclamped return
value exists only to decide when to retire a sample. A rebuild should decide deliberately
between true interpolation (render one interval behind the server, smooth, adds latency)
and extrapolation (predict past the newest sample); this code does neither, and its
visible behaviour is "jump to the server's last known pose", which is the honest
description of what multiplayer physics props look like in the shipped game.

## `PH_A_CrPr` / `PH_B_CrPr` / `PH_I_CrPr`

**Contract** — the three hooks around the network correction-and-prediction physics pass.
Only the *after* hook does anything, and only on the first pass following a spawn: it forces
the pose to agree with the shell, refreshes the object's spatial-index entry, then pins the
shell's first element and tells it to ignore static geometry.

**Invariants** — this runs only on a client (it asserts it is not the server). Pinning the
first element and ignoring statics turns a freshly spawned remote prop into something the
network correction drives directly, rather than something the local solver fights the
server over. It is a one-shot: `just_after_spawn` is cleared and never set again.

## `get_door_vectors`

**Contract** — recovers a door's closed and open swing directions from the model alone, for
the AI and for debugging. Returns false — meaning "this is not a door" — when the model has
no bone named `door`, when that bone's shape is not a box, when the shape is marked
physics-less, or when the bone's joint limits cover a full half-turn in both directions.

```text
FUNCTION get_door_vectors() -> optional<(closed, open)>
  bone = bone named "door" ; IF none THEN RETURN none
  shape = bone's shape ; REQUIRE it is a box and not marked physics-less
  pose  = object transform composed with the bone's animated transform
  axis  = the box's local x direction
  IF axis points away from the box's centre (measured from the pose origin)
      invert axis                                     # see note
  limits = the bone joint's second-axis angular limits
  IF the limit range spans a half turn in both directions THEN RETURN none
  open   = axis rotated about the vertical by the negated low limit
  closed = axis rotated about the vertical by the negated high limit
  transform both into world space
```

**Notes** — the axis flip makes the result independent of which way the door leaf was
authored. The box's local x is a line, not a direction; disambiguating it by the vector
from the hinge to the leaf's centre gives "outward along the leaf" every time.

The joint-type check was deliberately removed so that sliding doors — which use a different
joint kind but still want an open and closed direction — are described by this function too.
The remaining rejection is a joint with no meaningful limits, which cannot describe a swing.

## `Load` / `net_Save` / `net_Destroy` / `net_SaveRelevant` / `InitServerObject`

**Contract** — the lifecycle remainder, each one base behaviour plus the breakable-skeleton
layer's own. Destruction additionally re-arms the skeleton layer for respawn and releases
the bone-motion sound. `net_SaveRelevant` is unconditionally true: every physics prop's
pose is saved, because props are moved by the player and a save that reset them would be
visible. `InitServerObject` writes the live type back into the server record, so a saved
object reloads with the shape it was spawned as.

## `UsedAI_Locations` / `is_ai_obstacle`

**Contract** — a physics prop never occupies a navigation vertex (it is not a creature and
must not block pathfinding by presence), but by default it *is* an obstacle that the AI
avoidance layer routes around. The obstacle behaviour is overridable per configuration
section, because small props (a bucket) should be walked over rather than routed around.

## `get_collision_hit_callback` / `set_collision_hit_callback`

**Contract** — installs the callback notified when this body takes a collision hit. The
setter takes ownership and destroys whatever was installed before, so the callback's
lifetime is the object's. A rebuild expresses this as an owned optional.
