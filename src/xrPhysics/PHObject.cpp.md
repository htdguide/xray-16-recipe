# src/xrPhysics/PHObject.cpp

> The lifecycle and collision pass of one simulated unit: activation, freezing, deferred disabling, and the broadphase query that turns nearby objects into contacts.

**Needs** — [`PHObject.h`](PHObject.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PHIsland.h`](PHIsland.h.md) · [`PHMoveStorage.h`](PHMoveStorage.h.md) · [`Physics.h`](Physics.h.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`dRayMotions.h`](dRayMotions.h.md) · [`console_vars.h`](console_vars.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHObject.h`](PHObject.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md)
**Tier floor** — T2: scheduling and queries; no layout or device contact.

## Purpose

A physics object is simultaneously a member of three registries and this file keeps them
consistent: the world's active list (things that get stepped), the spatial index shared with the
rest of the engine (things that can be found by a box or ray query), and its own *island* (the set
of bodies the solver treats as one system this step).

The interesting decisions here are all about **when an object stops costing anything**. Physics is
the frame's worst tail, so the engine spends real complexity on four distinct kinds of "off":
active/inactive, frozen, recently-deactivated, and collision-disabled. They are not synonyms and
conflating them in a rebuild produces either jitter or objects that fall through the floor.

## State

```text
RECORD PhysicsObject
  flags            : set of {activated, frozen, dirty, net_interpolation,
                             traces_motion, recently_deactivated}
  island           : Island                # this object's own solver group
  collide_bits     : int (bit set)         # which collision groups this object answers
  collide_class    : int (bit set)         # which classes it belongs to
  check_count      : int                   # countdown before a deactivated object is forgotten
  half_extents     : vector                # box half-size used for the broadphase query
  spatial_sphere   : sphere                # bounding sphere kept in the engine's spatial index

  # invariant: activated and frozen are mutually exclusive
  # invariant: recently_deactivated implies not activated and not frozen
  # invariant: dirty is true from the moment the object moves until it has collided this step;
  #            an object only ever generates contacts against a *dirty* partner, which is how a
  #            pair is considered exactly once per step instead of twice
```

## `activate` / `deactivate`

**Contract** — `activate` puts the object into the world's stepped list; it refuses if the object
has no collision shape, is a no-op if already active, and *thaws instead of activating* if the
object is frozen. `deactivate` removes it. Both are forbidden while the world is mid-step, because
the world iterates that list.

**Invariants** — the island must be in its unmerged, self-owned state when either is called;
merging state exists only within a step.

```text
FUNCTION activate()
  REQUIRE collision_shape() EXISTS      # activating a destroyed object is a programming error
  IF activated THEN RETURN
  IF frozen THEN unfreeze(); RETURN     # freeze is a stack level above activation
  IF recently_deactivated THEN remove_from_recently_deactivated()
  world.add_to_active(self)
  on_processing_activate()
  activated := true
```

## `freeze` / `unfreeze`

**Contract** — freezing moves an object from the active list to the world's frozen list without
touching its state; thawing moves it back. Freeze exists so the whole world can be held still while
a single object is single-stepped (the character restriction resolver in
[`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) does exactly this), and so an object can be
parked without the cost of teardown.

**Notes** — `freeze_content` / `unfreeze_content` are the half that subclasses extend to put their
own bodies to sleep; `freeze` / `unfreeze` are the half that moves list membership. The split
exists because the world can freeze *every* object at once by moving the whole list in O(1) and
then calling only the content half on each.

## `put_in_recently_deactivated` / `check_recently_deactivated`

**Contract** — when an object falls asleep it is not immediately forgotten: it is parked on a
third list with a countdown. Each time the world's shared disable counter wraps, every parked
object decrements; at zero the object is told to release its cached collision results and is
dropped from the list.

**Notes** — the delay exists because a sleeping object is very likely to be woken again soon (a
crate nudged twice), and the cached triangle sets it holds are expensive to rebuild. The countdown
length is a tunable; there is no derivation for its default beyond "a few tens of steps".

## `collide`

**Contract** — generates every contact this object participates in, for one step. Runs in three
parts and leaves the object clean. Allocates nothing per call beyond the shared result vector,
which is reused across objects.

```text
FUNCTION collide()
  # Part 1 — swept motion, only for objects that asked for it
  IF traces_motion THEN
    FOR EACH traced_shape IN move_storage()
      (from, to) := traced_shape.segment_this_step()
      IF from IS the sentinel "never placed" THEN CONTINUE
      dir := to - from ; len := |dir|
      IF len < epsilon THEN CONTINUE           # no motion, the ordinary shape test suffices
      hits := spatial_index.query_ray(from, dir/len, len, PHYSICS_ONLY)
      FOR EACH other IN hits WHERE other != self AND other.dirty
        motion_ray := world.shared_motion_shape()
        motion_ray.bind(traced_shape, from, dir, len)
        generate_contacts(self, other, motion_ray, other.collision_shape())

  # Part 2 — ordinary dynamic-versus-dynamic
  collide_dynamics()

  # Part 3 — against the level's immovable triangle soup
  IF collide_validator.allows_static(self) THEN
    generate_contacts_against_level_mesh(collision_shape(), self)

  dirty := false        # every later object this step will now skip pairing with me
```

**Notes** — the swept-motion path exists because a fast small object (a thrown bolt, a bullet-like
prop) can pass entirely through a thin obstacle between two steps. Rather than shrink the timestep,
the engine builds a *ray shape* spanning the object's last and current placement and collides that
instead. Only objects that opt in pay for it; see `set_all_geom_traced` in
[`PHShell.cpp`](PHShell.cpp.md).

The `dirty` handshake is the whole reason a pair is not processed twice. Every object is marked
dirty when it moves and cleared at the end of its own collide; an object therefore only pairs with
partners that have not yet run. This makes the pass order-dependent but exactly-once, and it is why
the world must iterate its object list in a stable order for the determinism requirement to hold.

## `collide_dynamics`

**Contract** — queries the spatial index with this object's bounding box, and for every dirty
partner the collision-group filter admits, hands the pair to the contact generator.

```text
FUNCTION collide_dynamics()
  hits := spatial_index.query_box(spatial_sphere.center, half_extents, PHYSICS_ONLY)
  FOR EACH other IN hits
    IF other = self OR NOT other.dirty THEN CONTINUE
    IF collide_validator.pair_allowed(self, other) THEN
      generate_contacts(self, other, collision_shape(), other.collision_shape())
```

## `step` / `step_single` / `reinit_single`

**Contract** — a *single-object step*: collide, solve, and restore the world, used when one object
must be advanced outside the normal loop (placing a corpse, resolving an overlap). `step_single`
returns whether the object's island grew — that is, whether it touched anything — which the caller
reads as "still resolving".

```text
FUNCTION step_single(step) -> bool
  collide_dynamics()
  grew := island.gained_bodies()
  IF grew THEN
    island.solve(step)
    reinit_single()             # unmerge every island touched, drop the contacts
    recompute_bounds()
    collide_dynamics()
    grew := island.gained_bodies()
  reinit_single()
  RETURN grew

FUNCTION reinit_single()
  island.unmerge()
  FOR EACH other IN last_query_result: other.island.unmerge()
  clear last_query_result
  release every contact constraint and its feedback and effector storage
```

**Notes** — `step_prediction` is declared and empty. The intent recorded in the source is to step
one object forward, read the result, and roll the world back — a local prediction for network
reconciliation. It was never implemented; a rebuild should treat it as absent, not as a contract.

## `spatial_move` / `spatial_register`

**Contract** — refresh the bounds from the subclass, push them into the engine's shared spatial
index, and mark the object dirty so it participates in this step's pairing. `collision_disable` /
`collision_enable` unregister and re-register without changing simulation membership — an object
that still steps but that nothing can hit.

## `CPHUpdateObject`

**Contract** — a second, lighter participant in the step loop: something that wants the two
callbacks (`tune` before the solve, `data_update` after) without being a collidable object at all.
Breakable-shell bookkeeping and static geometry proxies ride on this. Activation is idempotent and
self-deregisters on destruction, because the world holds a raw link.

```text
FUNCTION activate()
  IF active THEN RETURN
  world.add_to_update_list(self)
  active := true
```

**Notes** — `net_relcase` is the escape hatch: when a shell is destroyed out from under the
simulation (a network correction deleting an entity), every update object is given the chance to
drop its reference before the memory goes. A rebuild with tracked references removes this.
