# src/xrGame/UIZoneMap.cpp

> The minimap: a rotating slice of the level's map texture under a fixed centre mark, with a compass, a clock and a contacts counter.

**Needs** — [`UIZoneMap.h`](UIZoneMap.h.md) · [`ui/UIMap.h`](ui/UIMap.h.md) · [`ui/UIHelper.h`](ui/UIHelper.h.md) · [`ui/UIInventoryUtilities.h`](ui/UIInventoryUtilities.h.md) · [`Actor.h`](Actor.h.md) · [`PDA.h`](PDA.h.md) · [`Level.h`](Level.h.md)
**Used by** — reached through its declarations in [`UIZoneMap.h`](UIZoneMap.h.md); callers name that, not this file.
**Tier floor** — T3: transform bookkeeping over a widget tree; the sub-level lookup is data-driven.

## Purpose

The minimap is a clipped window onto one large per-level texture. The map *moves and
rotates under* a fixed centre mark rather than the mark moving over a static map, so the
player's heading is always up. This file owns that coupling — camera position and heading
in, texture offset and rotation out — plus the three decorations that share the frame and
the rule that picks a different map image on levels built in vertical layers.

## State

```text
RECORD ZoneMap
  visible              : bool = true
  active_map           : MiniMap             # the rotating clipped map itself
  background, centre, compass, clip_frame : Static
  contacts_counter, contacts_counter_text : Static
  clock_text           : optional<Static>
  pointer_distance_text : optional<Static>   # single player only
  current_sub_map_index : int                # which vertical layer is displayed
```

**Invariant** — the compass and the map carry the *same* heading, negated from the
camera's, so a landmark drawn on the map and the compass bearing to it agree.

## `Init` — layout and the aspect correction

**Contract** — builds the widget tree from the minimap layout document: background, clip
frame, centre mark, an optional clock, and — only in single player — an optional
pointer-distance readout and a contacts counter. Allocates the map inside the clip frame
with heading enabled.

The interesting half is the `motionIconAttached` path, which is really "this heads-up
layout places the minimap next to the stamina/motion icon". In that layout the minimap
must be **square in screen space**, which the layout document cannot express because it
works in a virtual coordinate space that is not square. So the sizes are recomputed here:

```text
IF next_to_motion_icon THEN
  k = current horizontal-to-vertical aspect correction
  # take the frame's height as authoritative, derive width from it, so the map is round
  clip.height = clip.height * base_height * k
  clip.width  = clip.height / k
  clip.position = clip.position * base_height

  background.height = background.height * base_height
  background.width  = background.height * k
  background.position = centre of the clip frame

  # the compass, clock and counter were authored as FRACTIONS of the background,
  # so their positions are multiplied by the background's final size
  compass.position = compass.position * background.size
  clock.position   = clock.position   * background.size
  counter.position = counter.position * background.size

centre.position = clip.size / 2        # the mark sits at the frame's middle, always
```

**Notes** — in this path the authored positions of the compass, clock and counter are
*fractions of their parent*, while everywhere else in the UI they are absolute. That
reinterpretation is invisible in the layout document and is the single most surprising
thing in this file. A rebuild should make fractional positioning an explicit authored
attribute rather than a mode the code silently switches into.

## `Update`

**Contract** — no-op unless the view entity is the actor. Reads the camera, not the
actor's body, so the map tracks where the player is looking from.

```text
FUNCTION update(map)
  actor = the current view entity, as an actor; RETURN IF it is not one

  IF single player AND frame_number MOD 20 = 0 THEN
    n = actor.pda.active_contacts_count
    map.contacts_counter_text = n IF n > 0 ELSE empty

  map.clip_frame.update; map.background.update
  map.active_map.active_point = camera position
  IF map.pointer_distance_text exists THEN
    d = map.active_map.pointer_distance
    map.pointer_distance_text = d rounded to metres IF d > 0.5 ELSE empty

  heading = -(camera direction's heading)
  map.active_map.heading = heading
  map.compass.heading    = heading

  IF map.clock_text exists THEN
    map.clock_text = game clock, to the minute
```

**Invariants** — the contacts counter refreshes on every twentieth frame, not every
frame: counting active PDA contacts walks the contact list, and the number changes on a
human timescale. The pointer-distance readout blanks below half a metre rather than
showing "0 m".

## `Render`

**Contract** — draws the clip frame (and therefore the map inside it) and then the
background over it. Background *after* map is deliberate: the background is the bezel
that masks the map's square edges into a round window.

## `SetupCurrentMap`

**Contract** — called when a level finishes loading. Binds the map to the level's own
map texture, sets its working area to the clip frame's absolute rectangle, and computes
the zoom.

```text
FUNCTION setup_current_map(map)
  map.active_map.initialize(level name, the default heads-up map style)
  map.active_map.working_area = absolute rectangle of map.clip_frame

  zoom = map.clip_frame.width / 100
  IF the game configuration has a section for this level THEN
    zoom = zoom * that section's "minimap_zoom", if present
  ELSE IF the level's own configuration declares "minimap_zoom" THEN
    zoom = zoom * that value
  map.active_map.size = map.active_map.bounds * zoom
```

The divisor of 100 makes the zoom multiplier read as a percentage-ish number in the
shipped data; there is no other meaning to it. The per-level override is looked up in the
*game* configuration first and only then in the level's own, so a game-wide data pack can
retune a level's minimap without editing the level.

## `OnSectorChanged`

**Contract** — levels built in vertical layers (an underground complex beneath a surface
map) ship a `sub_level_map` table mapping a sector identifier to a map-image index. When
the camera's sector changes, this looks the new sector up; if it names a different index
than the one displayed, the map's texture is swapped to that numbered sub-image. Sectors
absent from the table leave the map alone, so only authored transitions switch layers.

**Notes** — the sentinel index (all bits set) means "the base image", and the code
computes a numbered sub-texture name before overwriting it with the base name in that
case; harmless, but a rebuild should branch first.

## `ZoomIn` · `ZoomOut`

**Contract** — accept the input and do nothing. Minimap zoom was cut; the entry points
survive because the input binding still exists. A rebuild should drop both and unbind.
