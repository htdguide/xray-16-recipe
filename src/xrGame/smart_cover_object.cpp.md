# src/xrGame/smart_cover_object.cpp

> Spawns a placed smart cover: builds its volume from the server record, registers the cover with the cover manager, then switches the entity off so it costs nothing for the rest of the level.

**Needs** — [`smart_cover_object.h`](smart_cover_object.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`Level.h`](Level.h.md) · [`xrServerEntities/xrServer_Objects_Alife_Smartcovers.h`](../xrServerEntities/xrServer_Objects_Alife_Smartcovers.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: shape construction at spawn, a point-in-volume test on demand

## Purpose

The bridge between an authored spawn record and a live cover. A smart cover object follows
the usual entity lifecycle only far enough to acquire a transform and a shape; then it
deliberately removes itself from every per-frame system in the engine. The cover it
registers outlives the frame and is consulted by the AI directly, so the entity itself has
nothing left to do.

The shape is not collision in the physical sense. It is the *extent* of the cover: the
volume a creature is considered to be inside, used by the planner and by scripts asking
whether someone is in a given cover.

## State

Declared in [`smart_cover_object.h`](smart_cover_object.h.md). The two enemy-distance
thresholds and the registered cover are all this object adds to a game object.

## `net_Spawn`

**Contract** — spawns the entity from its [server object](../../GLOSSARY.md) record.
Builds a composite shape from the record's list of spheres and boxes, runs the base
spawn, copies the two thresholds, registers a cover with the cover manager, and then
disables the entity. Reports but tolerates a record with no description. Returns whether
the base spawn succeeded.

```text
FUNCTION net_spawn(server_record) -> bool
  IF server_record.description is empty
    report "smart cover has no description"          # tolerated; see Invariants
  shape = new composite shape owned by this entity
  FOR EACH s IN server_record.shapes
    add s as a sphere or a box, per its type tag
  compute the shape's bounds
  IF base spawn fails                    RETURN false
  remove this entity from the AI-visible spatial class
  enter_min_enemy_distance = server_record.enter_min_enemy_distance
  exit_min_enemy_distance  = server_record.exit_min_enemy_distance
  IF the alife simulation is running AND the description is non-empty
    cover = cover_manager.add_smart_cover(description, self,
                                          is_combat_cover, can_fire,
                                          available_loopholes)
  ELSE
    cover = none
  stop processing
  disable
  make invisible
  RETURN true
```

**Invariants** —

- **The shape must exist before the base spawn**, because the base spawn computes the
  entity's spatial registration from its bounds. Building it afterwards registers a
  zero-extent object and the AI never finds the cover.
- **The cover is registered only when the [alife](../../GLOSSARY.md) simulation is
  running.** On a level loaded without it — the multiplayer and editor paths — smart
  covers are inert entities, because the job-assignment machinery that would use them is
  not there either.
- A missing description is a *report*, not a failure. The entity still spawns and still
  has a volume; it simply offers no cover. Shipped data contains such records, and failing
  on them would make levels unloadable.
- The entity clears its own AI-visibility spatial class explicitly. It has a shape, so the
  base spawn would otherwise put it in the set of things creatures can *see*, and creatures
  would react to an invisible box.
- The three shutdown steps at the end — stop processing, disable, make invisible — must all
  happen. Each removes the entity from a different system, and a rebuild that only hides it
  pays a per-frame cost for every cover on the level.

**Notes** — the record's combat and fire flags arrive as integers and are narrowed to
booleans here. The width is an artifact of the frozen spawn record format
(SYSTEM-REQUIREMENTS §5); only zero versus non-zero is meaningful.

## `Center` / `Radius`

**Contract** — the shape's bounding sphere centre transformed into world space, and its
radius. These are what the AI's spatial queries and the cover-point machinery use to find
this cover, so they must reflect the *shape*, not the entity's origin.

## `UpdateCL` / `shedule_Update`

**Contract** — both are unreachable. Reaching either means something registered this
entity with a per-frame or scheduled system despite the opt-outs, which is a bug in the
registration and not a condition to handle.

## `inside`

**Contract** — whether a world position lies within the cover's volume. Walks the composite
shape and answers on the first volume that admits the position.

```text
FUNCTION inside(position) -> bool
  FOR EACH s IN shape
    IF s IS a sphere
      IF distance(s.center, position) <= s.radius   RETURN true
    ELSE IF s IS a box
      build the box's eight world corners from the entity transform
      test the position against the six face planes
      IF admitted                                    RETURN true
  RETURN false
```

**Invariants** —

- The composite is a **union**: a position inside any one volume is inside the cover. That
  is what lets an author describe an L-shaped or multi-storey cover as several boxes.
- The sphere test compares against the sphere's centre **without applying the entity
  transform**, while the box test builds its corners *through* the transform. The two are
  therefore in different spaces, and a sphere volume on a rotated or translated smart cover
  tests against the wrong point. Shipped covers use boxes almost exclusively, which is why
  this has never mattered; a rebuild should transform both.
- The box test as written admits a position that lies on the inner side of **any one** of
  the six face planes, rather than requiring all six. Containment in a convex box is the
  conjunction, so the test as written accepts a large region outside the box. This is a
  defect, not a decision; a rebuild should require all six, and should expect the
  consequence that creatures will be judged "in cover" in fewer places than in the
  original.

## `OnRender` (development builds)

**Contract** — draws each volume of the composite shape, and for each loophole of the
registered cover a frustum standing for its field of view: the loophole's world eye point
and facing, its arc as the frustum's angle, and its range as the far plane. Draws nothing
beyond the shape when no cover is registered.

**Notes** — this is the authoring tool for smart covers. The arc and range a level designer
tunes are invisible in the data and only make sense seen in place, which is why the
visualization draws the *derived, world-space* geometry rather than the authored local
values.
