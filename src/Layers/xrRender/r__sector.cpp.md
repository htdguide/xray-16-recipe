# src/Layers/xrRender/r__sector.cpp

> Turning the level's authored portal polygons and sector records into the linked topology the visibility walk needs: a plane per portal, a bounding sphere, and the two-way wiring.

**Needs** — [`r__sector.h`](r__sector.h.md) · [`Common/LevelStructure.hpp`](../../Common/LevelStructure.hpp.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`FVF.h`](FVF.h.md)
**Used by** — [`r__sector.h`](r__sector.h.md)
**Tier floor** — T2: geometric setup at level load; nothing here is per-frame.

## Purpose

The level's visibility data ships as a list of portals — each a small polygon and the identifiers of the two sectors it separates — and a list of sectors — each a list of portal identifiers and the identifier of its root visual. This file derives from that the things the walk actually needs and cannot be stored: the portal's plane and bounding sphere.

## `Portal.setup`

**Contract** — Given one portal's authored record and the sector list, wire the portal and derive its geometry. Called once per portal at level load, after every sector exists. Fatal when the polygon is degenerate.

```text
FUNCTION setup(data, sectors)
  # Bounding sphere: the sphere of the polygon's bounding box. Not the minimal
  # enclosing sphere — a box sphere is cheap and the early test it feeds is
  # only a rejection, so looseness costs a little work and never correctness.
  sphere = bounding_box_of(data.vertices).bounding_sphere()

  polygon = data.vertices
  front   = sectors[data.sector_front]
  back    = sectors[data.sector_back]
  marker  = never

  # Plane normal: the average of the fan triangles' normals, skipping
  # degenerate ones. An authored portal is planar in intent but not to the
  # last bit, so a normal taken from one triangle would tilt slightly and the
  # side classifications at the polygon's far corners would be wrong. The
  # average is the robust choice.
  accumulated = zero ; contributing = 0
  FOR i IN 2 .. polygon.count - 1
    n = non_normalised_normal(polygon[0], polygon[i-1], polygon[i])
    IF n.length > epsilon
      accumulated += n / n.length ; contributing += 1
  FAIL WITH "invalid portal" IF contributing = 0
  plane = plane_through(polygon[0], accumulated / contributing)
```

**Invariants** — The plane passes through the polygon's first vertex, so the side classification is exact at that corner and approximate elsewhere by exactly the polygon's own non-planarity. The winding of the authored polygon determines which sector the normal points at, which is the *front*; the level compiler guarantees the correspondence and a rebuild must preserve it, because nothing here checks it.

## `Sector.setup`

**Contract** — Resolve the sector's portal identifiers to portal objects and its root identifier to a visual. Called once per sector at level load, after every portal object exists but before any portal is set up — which is why this resolves portals by identity only and the portals resolve sectors later.

On a dedicated server the root visual is left empty: the geometry is never loaded, and every later reader checks. Elsewhere the root is fetched from the renderer's visual table by identifier.

## `Portal.debug_render`

**Contract** — In a debug build, and only while the portal-display flag is set, draw the portal polygon twice: as a translucent blue fan from its centroid, then as a wireframe outline, with the depth comparison relaxed so the outline is not hidden by the wall it sits in. Registered with the frame loop at a low priority so it lands after the scene. Purely a diagnostic; a rebuild may omit it, but it is the only way to see the topology the level compiler produced.
