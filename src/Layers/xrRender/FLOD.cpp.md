# src/Layers/xrRender/FLOD.cpp

> Eight billboards around an object, an outward normal per billboard, and a screen-coverage factor derived from how much of the object's bounding sphere the object actually fills — with the draw itself removed.

**Needs** — [`FLOD.h`](FLOD.h.md) · [`xrCore/FMesh.hpp`](../../xrCore/FMesh.hpp.md) · [`R_DStreams.h`](R_DStreams.h.md)
**Used by** — reached through its declarations in [`FLOD.h`](FLOD.h.md); callers name that, not this file.
**Tier floor** — T1: it reads a frozen record as a byte image and declares a vertex layout.

## Purpose

The cheapest possible representation of a distant object: not geometry at all, but a photograph of it taken from one of eight directions. At a great enough distance a building or a tree becomes two triangles.

**The draw is commented out in the shipped source.** This type loads, computes its factor, and renders nothing. What survives here is therefore the *data* — which is still in every shipped level — and the two computations that are still live. The draw is documented from the disabled code because a rebuild that wants distant impostors needs it, and because it explains the data.

## The authored data — frozen

```text
RECORD ImpostorFacet          # eight per model, one per 45 degrees of azimuth
  corners : list<ImpostorVertex> of length 4
  normal  : vector3            # COMPUTED at load, not stored

RECORD ImpostorVertex
  position    : vector3
  texcoord    : vector2
  rgb_hemi    : int (32-bit, packed)   # baked static lighting, with the hemisphere
                                       #   (sky ambient) term in the fourth channel
  sun         : int (8-bit)            # baked sun visibility
```

**Invariants**

- Exactly eight facets, always. The count is not stored; it is structural.
- Each facet's normal is computed at load as the **average of the four normals of the quad's four corner-triples**, then normalized and **inverted**. Averaging four rather than taking one handles a non-planar quad, which the authoring tool does produce. The inversion makes the normal point *toward* the viewer position the billboard was rendered from, which is what the direction selection below needs.
- The lighting is baked per corner, and the hemisphere and sun terms are separate from the colour. That is the engine's universal lighting triple — static colour, sky visibility, sun visibility — and it appears identically on the tree and detail types.

## The level-of-detail factor

**Contract** — computed once at load: how much of the object's bounding *sphere* the object's bounding *box* actually occupies, seen from a typical angle.

```text
FUNCTION compute_lod_factor()
  r = the bounding sphere's radius
  (a, b, c) = the box's half-extents, SORTED ascending; take the middle one as `a`

  # The area of the circular segment of a disc of radius r cut at distance a,
  # doubled and doubled again — the object's silhouette area estimate:
  silhouette = 4 * 0.5 * (r*r * arcsin(a/r) + a * sqrt(r*r - a*a))
  disc       = pi * r * r
  factor     = silhouette / disc
```

**Invariants**

- The factor is a number between zero and one: how *solid* the object is within its bounding sphere. A cube nearly fills its sphere; a thin sign barely does. The draw stream multiplies a model's projected sphere area by this factor before comparing it against the switch threshold, so a thin object switches to its impostor sooner than a chunky one of the same bounding radius.
- The **middle** half-extent is used, not the largest or smallest. That is the estimate for "seen from a typical horizontal direction": the largest extent is usually height, the smallest is usually depth, and the middle one is the width that dominates the silhouette. It is a heuristic and it is the whole content of the choice.
- The formula is the exact area of a circular segment, computed twice and doubled — an odd arrangement that reduces to `2 * (r² asin(a/r) + a √(r² − a²))`. Written that way it is the area of the *lens* formed by two chords at ±a, which is the object's silhouette approximated as the sphere clipped to the box's middle width.

## The impostor draw, as it was

```text
FUNCTION render(command_list)
  direction = normalize(model centre - camera position)
  facet = the one of eight whose normal has the largest dot product with direction
  write its four corners into the shared dynamic stream, using the facet's normal
  draw two triangles through the shared quad index buffer
```

**Invariants** — The nearest facet by normal, with no blending. The disabled source carries two notes from its author naming what was missing: *smooth transitions* between adjacent facets, and *five-colouring* — blending across five of the eight rather than snapping to one. The drawn vertex record in the header is built for exactly that: it carries **two** positions, two normals, two texture coordinate pairs and two colours per vertex, so that a vertex program could interpolate between two adjacent facets. The record is complete; the code to fill it is not.

**Notes** — That is the honest state of this feature: the data ships, the vertex layout for the finished version ships, and the implementation was abandoned between the naive version and the intended one. A rebuild wanting distant impostors should implement the two-facet blend the vertex record was designed for, not the snapping version that was disabled.

## `load` / `copy`

**Contract** — load reads the container's children, then the eight facets, then declares the geometry over the shared dynamic vertex stream and the shared quad index buffer, then computes the factor. Copy shares the geometry declaration and the factor and copies the eight facets by value.

**Invariants** — The facets are copied **by value**, not shared, unlike every other model type's geometry. They are 400-odd bytes and there is no reason; it is an inconsistency, not a decision.
