# src/xrPhysics/debug_output.h

> The port physics draws and counts through when someone is watching, and the complete list of
> things that can be watched.

**Needs** — [`xrPhysics.h`](xrPhysics.h.md) · [`debug_output.cpp`](debug_output.cpp.md) · [`PHObject.h`](PHObject.h.md) · [`xrCDB/xrCDB.h`](../xrCDB/xrCDB.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHDebug.cpp`](../xrGame/PHDebug.cpp.md) · [`PHDebug.h`](../xrGame/PHDebug.h.md) · [`debug_output.cpp`](debug_output.cpp.md)
**Tier floor** — T3: an interface of drawing primitives and counters. It exists at all only
because physics must not reach the renderer directly.

## Purpose

Physics cannot draw. It has no access to the renderer, and
[the engine's module graph](../../SYSTEM-REQUIREMENTS.md#7-build-order) says it must not gain
any. Everything visual and every statistic physics wants to expose therefore goes through this
one interface, which the engine fills in when debug tooling is present and which is otherwise
a set of no-ops ([`debug_output.cpp`](debug_output.cpp.md)).

The whole file compiles away in a shipping build. That is the second decision, and it is the
reason the interface can afford to be as wide as it is: none of it costs anything when it is
gone.

**The list of draw flags below is the most useful thing on this page.** It is an inventory of
every phenomenon in this chapter that someone found hard enough to debug that they added a
switch for it — contacts, saved triangles, sign changes, mass centres, ladder state, ragdoll
blending, character control. A rebuilder reading it learns where the difficulty lies before
writing a line.

## State

```text
ENUM draw_flag                # first mask — thirty-two independent switches
  contacts                    # every generated contact point and normal
  enabled_aabbs               # the extent of every awake object
  intersected_tries           # triangles the query returned
  saved_tries                 # triangles retained from the previous step's query
  tri_trace                   # the swept path a shape took between steps
  negative_tries              # triangles whose plane the shape is behind
  positive_tries              # triangles whose plane the shape is in front of
  tri_test_aabb               # the box used to query the collision database
  tries_changes_sign          # triangles the shape crossed this step
  tri_point                   # the point at which the crossing happened
  explosion_pos               # where a blast was applied
  object_statistics           # per-object counters as text
  mass_centers                # every element's centre of mass
  death_activation_box        # the box a corpse is pushed out of walls with
  hit_application_points      # where hits landed
  cashed_tries_stat           # how well the triangle cache is performing
  car_dynamics                # vehicle state as text
  car_plots                   # vehicle state as graphs over time
  car_all_transmission        # every gear, not just the engaged one
  ladder                      # climbable-volume state
  explosions
  z_disable                   # objects suppressed from sleeping
  always_use_ai_ph_move       # force the AI movement path
  never_use_ai_ph_move        # forbid it
  disp_obj_collision_damage   # collision damage as it is computed
  ik                          # inverse kinematics for foot placement
  ik_goal
  ik_limits
  character_control           # the character controller's state machine
  ray_motions                 # the swept-motion probes
  track_object                # everything, for one named object only

ENUM draw_flag_2              # second mask — the overflow, and the animation side
  track_object                # (again, in the second mask; see the note)
  actor_restriction
  ik_off
  hit_anims
  draw_ik_limits, draw_ik_predict, draw_ik_shift_object,
  draw_ik_collision, draw_ik_blending

ENUM track_detail             # what to print for the ONE tracked object
  blends_in_part_0 .. blends_in_part_3     # the four animation blend slots
  blend_motion_name, blend_time, blend_amount, blend_mix_params,
  blend_flags, blend_state, blend_dump
```

**Invariants** — the two masks are separate thirty-two-bit sets and the first is full. The
original carries an explicit warning not to confuse them, which is the honest signal: a
rebuild with a flag *set* rather than two packed integers deletes the whole hazard.

**Notes** — `track_object` appears in both masks with different meanings, which is exactly the
confusion the warning is about. The tracked object is named by a string the interface exposes,
and the third enumeration says what to print about its animation blending — a physics debug
facility reaching into animation, because the hardest bugs in this chapter are precisely the
ones where an animation and a ragdoll disagree about who owns a bone.

## `IDebugOutput`

**Contract** — an implementor must provide four groups:

- **The two draw masks and the tracked object's name**, read every time physics considers
  drawing anything.
- **Drawing primitives**, all in world space, all taking a packed colour: a line, an
  axis-aligned box, an oriented box, a point with a size, a coordinate frame with a size and
  an alpha, a contact, and a triangle — the last in two forms, one from a collision-database
  query result and one from a triangle plus the vertex array it indexes into.
- **A retained-draw bracket**: open a cached draw, emit into it, close it with a lifetime in
  milliseconds. Physics runs on a fixed step and the frame does not, so a mark made during one
  step must survive until it has been seen; the bracket is how a step's drawing outlives the
  step. Plus a frame-start hook, a render hook and a clear.
- **Mutable counters**, handed out by reference so physics can increment them in place:
  triangles tested, triangles saved for active objects, total saved triangles, queries reused
  per step, new queries per step, bodies, joints, islands, contacts, and the collision-damage
  velocity currently being displayed.
- **Per-object step hooks** — before and after each of: the data update, the step, the tuning
  pass, and collision — so a tracked object's state can be snapshotted at every phase boundary.

**Notes** — the counters are returned by reference rather than incremented through a method
purely so the increment costs nothing in a hot loop. That is an optimisation whose shape a
rebuild need not copy; what must survive is *which* counters exist, because
`reused_queries_per_step` versus `new_queries_per_step` is how the triangle cache in
[`tri-colliderknoopc/dSortTriPrimitive.h`](tri-colliderknoopc/dSortTriPrimitive.h.md) is
judged, and that cache is the chapter's main performance decision.

The eight per-object phase hooks name the pipeline exactly: data update → tune → collide →
step. A rebuilder can read the physics frame's structure off this list alone.

## `ph_debug_output` / `debug_output`

**Contract** — one module-level slot holding the current implementation, and an accessor that
asserts it is set. Physics never holds a null check at a call site; it calls the accessor and
relies on the slot always containing at least the no-op implementation. That is the point of
the no-op: it removes a branch from several hundred call sites.
