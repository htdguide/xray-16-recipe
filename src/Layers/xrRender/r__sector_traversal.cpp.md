# src/Layers/xrRender/r__sector_traversal.cpp

> The visibility walk: from the camera's sector, recurse through every portal whose polygon survives the current frustum, clipping the frustum and the screen rectangle at each step; and fill the distant portals with ambient so the world does not end in a hole.

**Needs** — [`r__sector.h`](r__sector.h.md) · [`HOM.h`](HOM.h.md) · [`FVF.h`](FVF.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r__sector.h`](r__sector.h.md)
**Tier floor** — T1: the recursion runs per render context per frame over small inline polygon buffers and must not allocate; the fade pass writes vertices into a mapped device buffer.

## Purpose

The first and cheapest of the renderer's three visibility filters. It turns "where is the camera" into "these sectors, each seen through these clipped frusta and bounded by these screen rectangles", and everything downstream works from that.

The walk applies up to four additional tests at each portal, each switchable, and the *order* they are applied in is the design:

1. **Bounding sphere against the frustum** — three dot products, rejects most portals outright.
2. **Coverage** — the portal's apparent size; a doorway a hundred metres away leads somewhere not worth drawing.
3. **Frustum clip of the polygon** — exact, produces the clipped polygon the next frustum is built from.
4. **Occlusion map** — the expensive one, and it is applied last, against either the clipped polygon or the screen rectangle depending on which is available.

## `traverse`

**Contract** — Walk from a starting sector and leave the reached sectors, their clipped frusta and their screen rectangles in place. Advances the traversal marker so no clearing pass is needed. Writes into every sector it reaches and every portal it passes through.

```text
FUNCTION traverse(start, frustum, view_pos, xform, options)
  marker += 1
  remember options, view_pos, xform
  xform_to_screen = viewport_mapping composed with xform
  reached.clear()
  IF fading requested THEN fading.clear()

  # The root rectangle is the whole screen at depth zero.
  traverse_sector(start, frustum, ScreenRect(0,0 .. 1,1, depth = 0))

  IF scissoring requested
    FOR EACH reached sector
      merged_rect = union of its rects, with the NEAREST of their depths
```

The viewport mapping is the fixed half-scale-and-offset that takes clip space to the zero-to-one screen box, with y flipped. It is composed once per traversal.

## `traverse_sector` — the recursion

**Contract** — Record this sector and the route into it, then try every portal out of it. Recurses. The recursion depth is bounded by the level's topology, not by the frustum, so a level with a long chain of small rooms recurses deeply; the original accepts this.

```text
FUNCTION traverse_sector(sector, frustum, rect)
  # First arrival this traversal clears last frame's routes. Subsequent
  # arrivals append. This is why there is no per-frame reset over all sectors.
  IF sector.marker != marker
    sector.marker = marker
    reached.append(sector)
    sector.frusta.clear() ; sector.rects.clear()
  sector.frusta.append(frustum)
  sector.rects.append(rect)

  FOR EACH portal OF sector
    IF portal.marker = marker THEN CONTINUE     # already passed through

    # --- Which side are we going to? ---------------------------------
    IF portal.dual_render
      # The camera is standing in this portal: it has no meaningful side, so
      # take whichever sector is not the one we are in.
      next = portal.other_side_from(sector)
    ELSE
      next = portal.sector_behind(view_pos)
      IF next is this sector    THEN CONTINUE   # facing the wrong way
      IF next is the start sector THEN CONTINUE # would walk back into the start

    # --- 1. Cheap sphere rejection -----------------------------------
    IF NOT frustum.may_contain(portal.sphere) THEN CONTINUE

    # --- 2. Coverage --------------------------------------------------
    IF coverage testing requested
      to_portal   = portal.sphere.centre - view_pos
      coverage    = portal.sphere.radius^2 / to_portal.length_squared
      # Foreshortening: a portal seen edge-on shows nothing, so the estimate
      # is scaled by how squarely we look at it.
      coverage   *= abs(dot(portal.plane.normal, normalise(to_portal)))
      IF coverage < discard_threshold THEN CONTINUE

      IF fading requested
        IF coverage < fade_start THEN remember (portal, coverage) for the fade pass
        IF coverage < fade_end   THEN CONTINUE    # stop walking; the fade covers it

    # --- 3. Exact clip ------------------------------------------------
    clipped = frustum.clip_polygon(portal.polygon)
    IF clipped is empty THEN CONTINUE

    # --- 4. Scissor and occlusion -------------------------------------
    IF scissoring requested AND NOT portal.dual_render
      project every clipped vertex through xform_to_screen, divide by w,
        and take the bounding rectangle and the NEAREST depth
      IF the nearest depth is behind the eye (negative)
        # The portal straddles the eye plane; its projected rectangle is
        # meaningless. Inherit the parent's rectangle and fall back to the
        # exact polygon-based occlusion test.
        next_rect = rect
        IF occlusion testing requested AND the polygon is occluded THEN CONTINUE
      ELSE
        next_rect = intersection of the projected rectangle with `rect`,
                    carrying the projected depth
        IF next_rect is empty in either axis THEN CONTINUE
        IF occlusion testing requested AND the rectangle is occluded THEN CONTINUE
    ELSE
      next_rect = rect
      IF occlusion testing requested AND the polygon is occluded THEN CONTINUE

    # --- Recurse ------------------------------------------------------
    next_frustum = frustum_from_portal(clipped, portal.plane.normal, view_pos, xform)
    portal.marker      = marker
    portal.dual_render = false          # consumed
    traverse_sector(next, next_frustum, next_rect)
```

**Invariants**

- The portal marker is set *after* every test passes, not before. A portal rejected on one route may still be entered from another route with a wider frustum.
- `dual_render` is cleared as the portal is entered, so a dual portal is entered dual exactly once per traversal.
- The screen rectangle is intersected, never replaced: a doorway seen through a doorway is bounded by both.
- The two occlusion variants are not equivalent. The polygon test is exact and expensive; the rectangle test is conservative and cheap. Which one runs is decided by whether a meaningful rectangle exists, which is why the eye-plane case exists at all.
- The new frustum is built from the *clipped* polygon and the portal's plane, so it is the exact pyramid from the eye through the visible part of the opening — not the portal's bounding box, and not the old frustum narrowed.

**Notes** — The "would walk back into the start sector" rejection is subtle: without it, a camera in a corridor would re-enter its own sector through the far portal of an adjacent room and accumulate a second, wrong frustum for it. It is correct only because the start sector is always reached first, with the full frustum.

## `fade_render`

**Contract** — Draw an ambient-coloured fill over every portal the coverage test decided was too small to walk through. Called once after the traversal, only when portal fading is enabled. Writes into the shared dynamic vertex stream and issues one draw. Clears the fade list.

```text
FUNCTION fade_render()
  IF fading is empty THEN RETURN

  # Back to front, so overlapping distant openings composite correctly.
  sort fading by squared distance from view_pos, descending

  triangles = sum over portals of (vertex count - 2)     # fan triangulation
  buffer    = map_vertices(triangles * 3, stride)

  ambient = the weather system's current ambient colour, with zero alpha

  FOR EACH (portal, coverage) IN fading
    # Alpha ramps from transparent at the threshold where walking stops being
    # worthwhile to fully opaque at the threshold where walking stops
    # entirely. The band is the same one the imposter cross-fade uses, so a
    # portal fades out over the same distance its contents fade in.
    alpha  = clamp(round((1 - (coverage - fade_end) / (fade_start - fade_end)) * 255), 0, 255)
    colour = ambient with that alpha
    emit the polygon as a triangle fan from vertex zero, every vertex in colour

  unmap
  set world transform = identity, bind the portal-fade material and geometry
  disable face culling      # a portal is a two-sided polygon
  draw the triangles
  restore face culling
  clear fading
```

**Invariants** — The fill takes the *ambient* colour of the current weather, not black. The effect being simulated is a distant opening filled with the haze of the room beyond; black would read as a hole. That the alpha is zero in the base colour and substituted per portal is why one colour value serves all of them.

**Notes** — The fan triangulation is done on the processor rather than with an index buffer because the portal count is small and the polygons have at most six vertices; a shared index pattern would have to handle three different triangle counts.

## `debug_draw`

**Contract** — In a debug build, draw each reached sector's merged screen rectangle as a wireframe box in clip space, so the scissor rectangles can be inspected. The list it draws from is populated only when a line in `traverse` is uncommented; as shipped it is always empty.
