# src/xrGame/space_restrictor.cpp

> The client object for a restrictor entity: an invisible, non-simulated volume assembled from authored spheres and boxes, which registers itself with the level's restriction registry on spawn and answers exact containment queries against a lazily rebuilt world-space cache.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [`space_restrictor_inline.h`](space_restrictor_inline.h.md) · [`GameObject.h`](GameObject.h.md) · [`space_restriction_manager.h`](space_restriction_manager.h.md) · [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`CustomZone.h`](CustomZone.h.md) · [`RadioactiveZone.h`](RadioactiveZone.h.md) · [`ZoneCampfire.h`](ZoneCampfire.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — reached through its declarations in [`space_restrictor.h`](space_restrictor.h.md); callers name that, not this file.
**Tier floor** — T2: point-in-volume tests against a cached set of planes and spheres

## Purpose

A restrictor is the only entity in the game whose entire purpose is to *be a shape*. It is
never drawn, never updated, never scheduled and never collided with in the physical sense;
it exists so that the restriction subsystem, and the zones and smart terrains that derive
from it, have a named volume to point at.

This file is the base of a substantial family — every anomaly, every damaging zone, every
campfire and every smart terrain is a restrictor underneath — so the decisions here about
spawn ordering, AI visibility and containment caching are inherited by all of them.

## State

```text
RECORD SpaceRestrictor                   # extends the game object
  restrictor_type : int (8-bit)          # which of the six senses this volume has
  spheres         : list<sphere>         # world space; derived
  box_planes      : list<6 planes>       # world space, inward half-spaces; derived
  bounds          : sphere               # world space; derived
  actual          : bool                 # are the three derived fields in step with the transform?
```

**Invariants** — the three derived fields are a cache of the authored collision form under
the object's current transform, and `actual` is their validity. Any transform change clears
it; the next containment query rebuilds. Nothing else may clear or set it.

The authored form itself — the list of spheres and boxes — lives in the collision form,
built from the server object at spawn and never modified afterwards.

## Restrictor types

Six values, and the distinctions among them are load-bearing:

```text
ENUM RestrictorType
  default_none      # not a restrictor at all; the object is a volume for some other purpose
  default_out       # a level-wide PERMITTED region: everyone must stay inside it
  default_in        # a level-wide FORBIDDEN region: everyone must stay out of it
  none              # a restrictor, but one nobody is automatically restricted by
  in                # a forbidden region, named explicitly by the entities it restricts
  out               # a permitted region, named explicitly by the entities it restricts
```

The `default_` prefix means *automatic*: registering such a restrictor appends its name to
the level's corresponding default list and re-derives every restricted entity. The
unprefixed `in` and `out` are named explicitly in spawn records and by scripts. The value
comes from the server object and therefore from the shipped level data, so the set is
frozen.

`default_none` is the value that means "do not register at all" — it is the type carried by
the many restrictor-derived objects (most zones) that are volumes for their own purposes and
do not constrain movement.

## `net_Spawn`

**Contract** — bring the volume online from its server object. Builds the collision form,
runs the base spawn, strips AI visibility, disables and hides the object, and registers with
the level's restriction registry. Returns whether the base spawn succeeded; registration is
skipped entirely when it did not. Hard-fails if the server object is not a restrictor
record.

```text
FUNCTION net_Spawn(server_object) -> bool
  actual = false                                  # any cached geometry is now stale

  restrictor_type = server_object.restrictor_type

  form = new collision form
  FOR EACH primitive IN server_object.shapes
    add it to form as a sphere or an oriented box per its kind
  form.compute_bounds()
  set this object's collision form to form        # BEFORE the base spawn

  IF NOT base.net_Spawn(server_object)  RETURN false

  # A restrictor is normally invisible to creature vision: it is a region, not a thing,
  # and letting it enter the senses system would have every creature "see" the map's
  # fences. The exception is when creatures are configured to avoid anomalies, in which
  # case a damaging zone must be perceivable — but radiation zones and campfires stay
  # invisible even then, because they are survivable and routing around them is wrong.
  IF creatures do not avoid anomalies
     OR this is not a damaging zone
     OR this is a radiation zone OR a campfire
    clear the AI-visible flag on this object's spatial record

  disable this object                             # no physics, no collision response
  hide this object                                # never drawn

  IF the level has no navigation mesh             RETURN true
  IF restrictor_type IS none                      RETURN true
  level.restriction_manager.register_restrictor(self, restrictor_type)
  RETURN true
```

**Invariants** — the ordering is load-bearing at three points.

1. The collision form is built and installed **before** the base spawn, because the base
   spawn computes the object's spatial record from its bounds.
2. Registration happens **after** the base spawn, because it reads the object's name and
   transform, and because it immediately builds a border from the collision form under that
   transform.
3. The navigation-mesh check comes before registration: a level with no mesh — a multiplayer
   map, or one shipped without AI data — has no vertices for a border, and building one
   would fail rather than degrade.

**Notes** — disabling and hiding are what make this a pure volume. Enabled would put it in
the physics world; visible would put it in the render list. Neither is wanted, and the
derived zone classes that do need an effect re-enable only what they need.

## `net_Destroy`

**Contract** — unregister from the restriction registry, after the base teardown. Skipped
when the level has no navigation mesh or the type is `none` — exactly the conditions under
which registration was skipped.

**Invariants** — the skip conditions must mirror the spawn path's exactly, or the registry
is asked to unregister a name it never registered, which is a hard failure. A rebuild should
derive both from one predicate rather than repeat the condition.

The base teardown runs **first**, the opposite order from spawn. Unregistering swaps a
placeholder into the bridge under this restrictor's name, which makes every restriction
naming it inert — so anything still querying during teardown gets "unrestricted" rather than
a reference to a half-destroyed object.

## `inside`

**Contract** — is a sphere within this restrictor's volume? Rebuilds the world-space cache
if stale, rejects on the bounding sphere, then tests the primitives. Const in intent though
it mutates the cache. Allocates only when the cache is rebuilt.

```text
FUNCTION inside(query) -> bool
  IF NOT actual  prepare()
  IF NOT bounds.intersects(query)  RETURN false
  RETURN prepared_inside(query)

FUNCTION prepared_inside(query) -> bool
  FOR EACH s IN spheres
    IF s.intersects(query)  RETURN true
  FOR EACH b IN box_planes
    # Inside a box, with the query radius as slack on every face.
    IF signed_distance(p, query.centre) <= query.radius FOR ALL 6 planes p IN b
      RETURN true
  RETURN false
```

**Invariants** — the volume is the **union** of its primitives, which is what lets an
authored region be assembled from overlapping parts. The bounding-sphere reject is exact
enough to be safe because it is computed to enclose every primitive.

The query radius is slack on the box test rather than a true sphere-box distance, so a
sphere near a box *corner* is admitted slightly early — the test treats the corner region as
if it were rounded outward. At the radii this is called with (half a navigation cell, or a
near-zero epsilon) the error is far below a cell, and the exact test would cost a distance
to the nearest feature instead of six dot products.

## `prepare`

**Contract** — rebuild the world-space cache from the authored collision form under the
current transform, and mark it valid. Allocates the two lists.

```text
FUNCTION prepare()
  bounds = sphere(transform applied to the form's centre, the form's radius)
  spheres = empty ; box_planes = empty
  FOR EACH primitive IN the collision form
    IF it is a sphere
      APPEND sphere(transform applied to its centre, its radius)   # radius unscaled
    ELSE
      corners = the 8 unit-cube corners through (object transform x box transform)
      APPEND the 6 planes built from corner triples, oriented inward
  actual = true
```

**Notes** — a sphere's radius is carried through unscaled while a box is rebuilt from its
transformed corners. That is an asymmetry, not a subtlety: an authored sphere under a scaled
object transform would be wrong, and the shipped data never scales a restrictor, so it has
never mattered. A rebuild should either scale the radius or state the no-scale assumption.

Converting boxes to six inward planes up front is what makes the containment test six dot
products instead of a transform into box space per query. Containment is asked once per
navigation vertex over a whole region during border construction, and repeatedly afterwards
by accessibility queries, so the conversion pays.

## `spatial_move`

**Contract** — the object's transform changed. Run the base handling, then invalidate the
cache.

**Invariants** — this is the *only* invalidation. A rebuild that caches world geometry
anywhere else in this family must hook the same point, and must remember that the border
built at spawn is **not** invalidated here — see
[`space_restriction_shape.cpp`](space_restriction_shape.cpp.md).

## `Center`, `Radius`

**Contract** — the world-space centre and the radius of the collision form's bounds. These
are the object's bounds for every purpose, including the bounding sphere a composition
unions.

## `UsedAI_Locations`

**Contract** — constantly false: a restrictor does not occupy a navigation vertex. It is a
region *over* the mesh, not an obstacle *on* it, and claiming vertices would make the
pathfinder route around the region rather than be constrained by it.

## `register_schedule`

**Contract** — constantly false: a restrictor is never given an update slice. It has no
behaviour; it only answers questions. This is what makes it affordable to have hundreds per
level.

## `OnRender` (checked builds)

**Contract** — draw every authored primitive as a wire ellipse or oriented box, coloured by
whether a derived zone is currently enabled, and print the object's name — and, for a zone,
its state — in screen space when the camera is within 100 metres. Gated behind a debug draw
flag. Debug-only.

**Notes** — the distance cut-off keeps the label pass from projecting every restrictor on the
level each frame; the value is a comfort threshold, not a derived one.

## Could not recover

The flag that decides whether damaging zones stay visible to creature vision defaults to
off, so in the shipped configuration creatures do not perceive anomalies at all and walk
into them. Whether that is the original behaviour being preserved or a modification left
disabled is not determinable from the source.
