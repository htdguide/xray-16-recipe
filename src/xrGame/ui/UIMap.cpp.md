# src/xrGame/ui/UIMap.cpp

> The four map surfaces and the one coordinate transform they share: world metres to canvas
> units, through a bound rectangle read from the level's own configuration.

**Needs** — [`UIMap.h`](UIMap.h.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`map_location.h`](../map_location.h.md) · [`map_manager.h`](../map_manager.h.md) · [`map_spot.h`](../map_spot.h.md) · [`Level.h`](../Level.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIMap.h`](UIMap.h.md)
**Tier floor** — T2: one specialised primitive submission for the round minimap; everything
else is arithmetic and widget manipulation.

## Purpose

Everything the game draws on a map goes through here. The file's real content is a coordinate
system and three variations on it, not four widgets: a map is a picture whose *canvas
rectangle* stands for a *world rectangle*, and every position the game hands it — a creature,
a quest marker, the player — is in world metres and must land in the right place at the
current zoom, the current pan and, on the minimap, the current heading.

Each map is a picture widget, which is why a spot can simply be a child: the picture's
position is the pan, and a child's position is relative to it.

## State

```text
RECORD CustomMap EXTENDS Static
  name          : text        # the level's name, or "global_map"
  bound_rect    : Rect        # the world region this picture covers, in metres
  working_area  : Rect        # absolute canvas rect to clip against; owned by whoever
                              # placed the map, not by the map
  locked        : bool        # suppress the spot refresh entirely
  rounded       : bool        # draw and clip as a disc rather than a rectangle
  pointer_dist  : real        # scratch: reset every frame, see below
  texture, shader_name : text

RECORD GlobalMap EXTENDS CustomMap
  map_wnd   : MapWnd
  min_zoom  : real            # 1 — the whole world fits the view
  max_zoom  : real            # from configuration

RECORD LevelMap EXTENDS CustomMap
  map_wnd     : MapWnd
  global_rect : Rect          # where this level sits on the world map, in world-map units
```

**Invariants**

- **The zoom is read from the width ratio only.** Both components of the zoom read-out are
  computed, but every conversion uses the horizontal one for both axes. A map is therefore
  always drawn at its bound rectangle's aspect, and a rebuild that honours both components
  will distort every map whose picture is not sized to its bounds.
- A map's own rectangle always starts at the canvas origin and is exactly the size of its
  bound rectangle at load; zoom and pan are expressed by later resizing and moving it.
- The working area is *absolute* while a map's own rectangle is relative to its parent. Every
  visibility and pointer decision converts one into the other explicitly, and getting that
  backwards is the classic mistake here.
- The scratch pointer distance is reset at the head of every update and written during the
  spot refresh. It is a one-frame channel between a map and the spots it is refreshing, not
  state.

## `Initialize` — where a map's geometry comes from

**Contract** — bind a map to a level by name. If that level is the one currently loaded, use
its already-parsed configuration; otherwise open that level's own configuration file from the
levels root and throw it away afterwards. Read the geometry from its `level_map` section if it
has one, and otherwise from a shared default section, warning that the level shipped no map.

**Notes** — the default fallback is what lets a level with no authored map still appear: it
gets a placeholder texture and a symmetric ±10000-metre bound. A rebuild must keep the
fallback, because user-installed levels routinely lack the section.

## `Init_internal` — reading the bound rectangle

**Contract** — record the name; take the texture and the bound rectangle from the named
section, each overridable by a same-named key in a section named after the map itself; apply
the horizontal canvas correction to the bound rectangle's two horizontal edges *when the map
does not rotate*; size the widget to the bound rectangle; place it at the origin; bind the
texture with the given shader; and mark it stretched.

```text
FUNCTION init_internal(name, config, section, shader)
  texture     <- config.get(section, "texture") OR "ui/ui_nomap2"
  IF config HAS (name, "texture") THEN texture <- that          # per-map override
  bounds      <- config.get(section, "bound_rect") OR (-10000,-10000,10000,10000)
  IF config HAS (name, "bound_rect") THEN bounds <- that

  IF this map does not rotate THEN
    bounds.left  <- bounds.left  * horizontal_canvas_correction
    bounds.right <- bounds.right * horizontal_canvas_correction
  size <- bounds.size;  position <- origin
  bind_texture(texture, shader);  stretch <- true
```

**Notes** — the aspect correction is folded into the *bounds* rather than applied at draw
time. A non-rotating map is drawn axis-aligned, so pre-scaling its world extent makes every
later conversion aspect-correct for free; a rotating map cannot do that, because the rotation
must happen in square units and the correction is applied after it instead. This split is the
reason two of the three conversions below exist.

## The conversions

**Contract** — three functions, and which one applies is decided by whether the map rotates.

- *World to local, untransformed* — the pure mapping: subtract the bound rectangle's origin,
  scale by the zoom, and **flip the vertical axis**, because world coordinates grow northward
  and canvas coordinates grow downward.
- *World to local* — the untransformed mapping plus whatever the map needs: for a non-rotating
  map, undo the aspect correction baked into the bounds, convert, then re-apply it to the
  result; for a rotating map, convert, rotate about the heading pivot, and apply the aspect
  correction *inside* the rotation — but only when the result is for drawing, not when it is
  for hit-testing.
- *Local to world* — the inverse, used to answer "what place did the player click".

```text
FUNCTION world_to_local_no_transform(p, bounds) -> Point
  zoom <- widget.width / bounds.width
  RETURN ( (p.x - bounds.left) * zoom,
           (bounds.height - (p.y - bounds.top)) * zoom )     # north-up flip

FUNCTION world_to_local(p, for_drawing) -> Point
  IF NOT rotating THEN
    b <- bounds WITH horizontal edges divided by the canvas correction
    r <- world_to_local_no_transform(p, b)
    RETURN r WITH x multiplied by the canvas correction
  ELSE
    pivot <- the heading pivot                                # the player's position on the map
    r <- world_to_local_no_transform(p, bounds) - pivot
    r <- rotate(r, by: heading,
                then scale x by: for_drawing ? canvas correction : 1)
    RETURN r + pivot
```

**Notes** — the *for drawing* flag is the load-bearing detail. A rotated minimap is drawn with
its horizontal axis stretched, so a drawn position must be stretched too; but when the same
conversion answers a geometric question — how far is this spot from the centre — the stretch
would corrupt the distance. One conversion, two consumers, one flag. A rebuild that keeps
canvas units square throughout does not need the flag and should say so.

Note also that the world map overrides this entirely: its bound rectangle is already in canvas
units, so its conversion is a plain scale with **no vertical flip**. Level maps sit on the
world map, not in the world, which is why their placement rectangle is in world-map units.

## `GetPointerTo` — marking something that is off the map

**Contract** — given a position in the map's local units and the radius of the marker, produce
the position on the edge of the visible area where an arrow pointing at it should sit, and the
heading that arrow should carry. Fails when the map and the working area do not overlap at
all, or when the position is inside the visible area.

```text
FUNCTION pointer_to(target_local, marker_radius) -> optional<(position, heading)>
  IF working_area DOES NOT INTERSECT map's absolute rect THEN RETURN none
  clip <- working_area rebased into the map's local units
  dir  <- normalize(clip.center - target_local)
  hit  <- first intersection of the ray (target_local, dir) with clip's border
  IF none THEN RETURN none
  heading  <- the compass heading of -dir
  position <- hit pushed a marker radius further along dir, floored to whole units
  RETURN (position, heading)
```

The rounded minimap replaces this with the obvious polar form: the arrow sits on the circle of
the working area's half-width, in the direction of the target from the heading pivot, with the
horizontal axis stretched by the canvas correction.

**Notes** — the pushed-out step reuses the intersection point as both the base and the scale
of the offset, which multiplies rather than adds. The visible effect is that markers far from
the centre of the canvas are pushed further out than markers near it. That is a bug preserved
here because the shipped minimap is small and centred, where the difference is invisible.

## Fitting and framing

**Contract** — `FitToWidth` and `FitToHeight` resize the map to the given extent, preserving
the bound rectangle's aspect. `OptimalFit` picks whichever of the two makes the map fit
entirely inside a rectangle: fit to height when the map is relatively taller than the target,
otherwise to width. `SetActivePoint` pans the map so that a world position lands at the centre
of the working area, and sets the heading pivot to that position — so the point the map is
centred on is also the point it rotates about. A position outside the bounds is ignored.

## Visibility predicates

**Contract** — `IsRectVisible` answers whether a local rectangle, rebased to absolute,
intersects the working area. `NeedShowPointer` answers whether it *fails* to intersect the
working area shrunk by five units on each side. The rounded minimap answers both by comparing
distances from the heading pivot against the working area's half-width, treating every spot as
a circle of its own half-width.

**Notes** — the five-unit inset means a marker that is only just visible still gets an
off-screen arrow, so an arrow and its target never appear at the same place with the arrow
suddenly vanishing. The two predicates are deliberately not each other's negation.

## `Update` and `Draw` (base)

**Contract** — the update resets the scratch pointer distance, refreshes the spots unless the
map is locked, and then updates as a picture. The draw pushes the working area as a scissor
rectangle, draws as a picture, and pops it. Locking is how the world map freezes every level
map's spot set while an animated zoom is in flight.

## The world map

**Contract** — initialises itself from a fixed configuration section, reads its maximum zoom
from the section named after itself, starts hidden with a minimum zoom of one, and detaches
every child map's spots at the head of each update — the level maps rebuild them immediately
afterwards. Pointer movement with no button held asks the owning screen to show the level's
name as a hint. Panning clamps.

### `ClipByVisRect` — the pan clamp

**Contract** — after any pan, move the map so that no edge of the working area shows past it:
its top-left may not be positive, and its bottom-right may not fall short of the working
area's extent. A map smaller than the working area is therefore pinned to the working area's
far corner.

### `CalcOpenRect` — where a zoom animation is going

**Contract** — given a point on the world map in *identity* zoom units and a target zoom,
produce the rectangle the map will occupy at that zoom with the point centred in the visible
area and the pan clamp applied, and return the distance the map's centre will travel in
identity units. The caller uses that distance to set the animation's duration, so a long
journey takes proportionally longer.

```text
FUNCTION calc_open_rect(center_identity, target_zoom) -> (rect, travel)
  rect   <- bounds.size scaled by target_zoom, at the origin
  centre <- center_identity scaled by target_zoom
  view   <- the owning screen's active map rectangle
  rect   <- rect moved so that centre sits at the middle of view
  rect   <- rect moved again by the same clamp the pan uses
  travel <- distance between the centres of (current rect / current zoom)
                                        and (rect / target zoom)
```

**Notes** — both centres are divided back into identity units before the distance is taken, so
the returned travel is comparable across zooms. That is the whole point of the function: an
animation's speed must be expressed in world terms, not in pixels, or zooming in makes every
pan feel slower.

## The level map

**Contract** — a level map recomputes its own rectangle every update by converting its
placement rectangle through the world map's conversion, so panning or zooming the world map
moves every level map with it and no level map holds a position of its own. It shows the
level's name as a hint after the cursor has dwelt on it for half a second — but only while the
world map is fully zoomed out, since at any other zoom the name is drawn on the map itself. It
forwards its spots' four notifications — show hint, hide hint, select, select-alternate — to
the owning screen. Pointer movement with no button held behaves as on the world map, and every
pointer action is swallowed while the world map is locked.

### `UpdateSpots`

**Contract** — drop every spot, and if this level map overlaps the screen's active map area,
ask each of the map manager's actual locations *that belongs to this level* to re-place itself
on this map. A location does the placing, not the map.

**Notes** — spots are rebuilt from scratch every frame rather than diffed. That is affordable
because the set is small and because a location's position changes continuously; it also means
a spot holds no per-frame identity, which is why the hint and selection notifications carry
the spot itself.

### `Draw` — spot scaling, and the three games' three rules

**Contract** — before drawing, resize every child spot that is marked scalable according to
the world map's current zoom, and hide every non-scalable spot whose lower scale bound the
zoom has not reached. The three shipped games disagree about the rule:

```text
z <- world map's current zoom
FOR EACH spot IN children
  IF spot.scalable THEN
    IF game IS the first game        THEN size <- origin_size * z
    ELSE IF game IS the second game  THEN
      IF z is between the spot's bounds THEN
        size <- origin_size * ((z - low) / (high - low))    # fades in across the band
      ELSE IF z is above the high bound THEN size <- origin_size
      # below the low bound: the size is left at whatever it was
    ELSE                                   # the third game
      size <- origin_size * clamp(z, low, high)
  ELSE IF spot.low_bound > 0 THEN
    spot.visible <- spot.low_bound < z
```

**Notes** — this is the sharpest surviving example of *shipped data deciding behaviour*. The
three rules are not refinements of each other: the first scales without limit, the second uses
the bounds as a fade-in window and leaves the spot untouched below it, the third uses them as
a clamp. Which runs is selected by which game's data is mounted. A rebuild must keep all three
or accept that markers on two of the three games look wrong at extreme zooms. The second
game's below-bounds case leaving the size alone is almost certainly unintended, and it is
observable: a spot retains the size it had when the zoom last crossed into the band.

### `CalcWndRectOnGlobal`

**Contract** — the level map's placement rectangle converted into the world map's *parent*
space, i.e. including the world map's own pan. Used to answer where on screen a level sits.

**Notes** — the placement rectangle's aspect must match the bound rectangle's, or the level
map is stretched relative to its picture; a development build warns and suggests the corrected
edge. The check ships disabled, so a mismatched user level is silently distorted.

## The minimap

**Contract** — rounded by default, tinted to half opacity at load, and refreshing its spots
from *every* location regardless of level — the minimap shows what is near, and nearness is
decided by the location, not by this file. When not rounded it behaves exactly as the base.

### `Draw` — the round map

**Contract** — when rounded, the picture is not drawn as a quad. It is emitted as a
twenty-segment polygon fan approximating a disc, centred on the working area, with the
positions rotated by the map's heading and the texture coordinates *not* rotated — so the map
turns under a fixed circular window. The horizontal axis is stretched by the canvas
correction on the position side only. Children are drawn normally afterwards.

```text
FUNCTION draw_round()
  n <- 20                                    # segments; a fixed approximation of the circle
  radius_px <- working_area.width / 2
  centre    <- working_area.centre
  radius_uv <- radius_px / widget.width      # the same radius in texture units
  FOR i IN 0..n-1
    a <- i * 2*PI / n
    position[i] <- centre + (radius_px * cos(a + heading) * canvas_correction,
                            -radius_px * sin(a + heading))
    texcoord[i] <- heading_pivot_in_uv + (radius_uv * cos(a),
                                         -radius_uv * sin(a) * widget_aspect)
    position[i] <- position[i] scaled to physical pixels
  emit a fan of n-2 triangles over those vertices
```

**Notes** — three things here break the chapter's normal rules, and all three are deliberate.

- **The vertices are converted to physical pixels inside this function.** Everywhere else in
  the toolkit that conversion happens once, at submission. Here it cannot, because the fan is
  submitted directly rather than through the quad emitter. A rebuild that routes this through
  a general polygon emitter removes the special case; keeping the picture's own scale factors
  is what matters, not where the multiply happens.
- **The disc is clipped by geometry, not by a scissor rectangle.** The base map pushes a
  scissor; the round one simply does not draw outside the circle. That is why the minimap is
  the only map whose visibility predicates are distance comparisons.
- **Twenty segments is not a quality knob.** At the shipped minimap's size the polygon's
  deviation from a circle is under half a canvas unit. A rebuild may raise it freely.
