# src/Layers/xrRender/r__sector_detect.cpp

> Answering "which sector is this point in", by casting a ray and asking whichever surface it hits first.

**Needs** — [`r__sector.h`](r__sector.h.md) · [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`xrEngine/IGame_Level.h`](../../xrEngine/IGame_Level.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`r__dsgraph_structure.h`](r__dsgraph_structure.h.md) · [`r__sector.h`](r__sector.h.md)
**Tier floor** — T2: two ray queries and a comparison; the tier is fixed only by the collision database's interface.

## Purpose

Every moving thing must know which sector it is in, because the visibility walk only reaches objects in sectors it reached. Sectors are not stored as volumes — there is no point-in-sector test — so membership is answered indirectly: cast a ray and read the sector off whatever it hits.

The two sources of an answer are the level's triangles, each of which carries the sector it bounds, and the portal polygons, which say which sector faces a given side.

## `detect_sector(point)`

**Contract** — The sector containing a point, or nothing when the point is outside the topology. Blocks on up to four ray queries. Thread-safe against other queries; the collision database is immutable and the query context is per render context.

```text
FUNCTION detect_sector(point) -> optional<sector_id>
  # Look DOWN first. Almost everything stands on a floor, and a floor triangle
  # is the most reliable carrier of a sector identifier.
  result = detect_sector(point, direction = down)
  IF result is none
    # Nothing below: the point is under the world, or over a hole. Look up.
    result = detect_sector(point, direction = up)
  RETURN result
```

## `detect_sector(point, direction)`

**Contract** — The same, along a given direction.

```text
FUNCTION detect_sector(point, direction) -> optional<sector_id>
  # Two independent nearest-hit queries, against two different models.
  portal_hit   = nearest_hit(portal_model,  point, direction, within 500 metres)
  geometry_hit = nearest_hit(static_model,  point, direction,
                             within portal_hit's distance, or 500 if none)

  IF neither hit THEN RETURN none

  # Whichever is nearer wins; a tie goes to the portal.
  IF the portal hit is nearer (or equal within a small epsilon)
    # A portal knows both its sides. Take the side FACING the query point —
    # looking down through a doorway's portal from above means the point is on
    # the portal's upper side, and that is the sector it belongs to.
    RETURN portal_of(portal_hit.triangle).sector_facing(point)

  # An ordinary triangle carries its sector directly.
  RETURN sector_of(geometry_hit.triangle)
```

**Invariants**

- The portal query is run first and its distance caps the geometry query, so the geometry query stops early when a portal is nearer. That is a cost optimisation, not a correctness one; the comparison afterwards would give the same answer either way.
- The tie goes to the portal *with an epsilon of slack*, because a portal polygon is normally coplanar with a doorway's geometry and floating-point order would otherwise decide arbitrarily. The portal's answer is the better one — it is authored specifically to disambiguate.
- The range of five hundred metres is a cap, not a tuning knob: it bounds the query's cost for a point in open space above a level. A point higher than that above any surface is reported as outside the topology, which is correct — it is.

**Notes** — The portal model is a separate collision tree built from the portal polygons alone. It exists only for this query. A level with no portals at all has no such model, and the function then relies entirely on triangle-carried sectors.

The whole approach means a point inside solid geometry, or in a sector whose floor is missing from the collision model, resolves to nothing and the object is dropped from rendering entirely. That is the failure mode a rebuilder will see as "objects invisible in one room", and the cause is always the level's data rather than this code.
