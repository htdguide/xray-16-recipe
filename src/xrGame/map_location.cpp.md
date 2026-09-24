# src/xrGame/map_location.cpp

> One marker on the map: where its subject is, whether that is still worth showing, and how it is drawn on the level map, the minimap and the edge-of-screen pointer.

**Needs** — [`map_location.h`](map_location.h.md) · [`map_spot.h`](map_spot.h.md) · [`map_manager.h`](map_manager.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`ui/UIMap.h`](ui/UIMap.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`GameTask.h`](GameTask.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`Inventory.h`](Inventory.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`location_manager.h`](location_manager.h.md) · [`level_changer.h`](level_changer.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md)
**Used by** — [`map_location.h`](map_location.h.md)
**Tier floor** — T3: derives screen geometry from world state, per frame

## Purpose

The map is not a separate data structure — it is a projection of the live world. A marker is
the thing that performs that projection for one entity: it finds where the entity is (which
may mean asking the alife simulation, because the entity may be offline on another level),
decides whether it should be visible at all, converts the position into map coordinates, and
attaches the right widget to the right map.

Its subtlety is that *everything* it shows is derived and everything is recomputed every
frame, so the file is organized around a one-frame cache and the question "is this marker
still meaningful".

## State

```text
RECORD MapLocation
  flags         : set of { serializable, hide_offline, has_ttl, position_tracks_actor,
                           pointer_enabled, spot_enabled, collidable, hint_enabled,
                           user_defined }
  hint          : text                 # string-table identifier, not display text
  object_id     : int (16-bit)         # the subject
  owner_record  : optional<ServerObject>  # the subject's authoritative record; none in multiplayer
  owner_task_id : text                 # the task this marker belongs to, if any

  level_spot, minimap_spot, complex_spot           : optional<Widget>
  their pointers                                   : optional<Widget>
  their active and inactive borders                : optional<Widget>
  border_names  : six text values      # the border widget descriptions, by map and state

  ttl           : int (seconds)        # 0 means no expiry
  expiry_time   : int                  # engine clock, milliseconds

  position_global : (real, real, real)     # this frame's world position
  position_on_map : (real, real)           # this frame's position in the current map's space

  cache:
    updated_frame : int                # the frame this cache was filled
    graph_vertex  : int                # the subject's coarse position, to detect level change
    position      : (real, real)       # world x and z; the map is a top-down projection
    direction     : (real, real)
    level_name    : text
    actual        : bool
```

**Invariant** — the cache is filled **at most once per frame** and every consumer reads it
rather than recomputing. Both maps and the pointer ring ask the same marker in the same frame,
and each ask would otherwise repeat an alife lookup and a graph query. The frame number is the
guard; a second update in the same frame returns the cached answer unchanged.

**Invariant** — a map position is world **x and z**. The map is a top-down projection and
height is discarded everywhere. This is assumed by every conversion in the file and by the
map widgets.

**Invariant** — `actual` means "this marker still describes something". A marker becomes
non-actual when its time to live expires or when its subject can no longer be found on either
side. Non-actual markers are not drawn and are swept by the manager; they are not deleted here.

**Invariant** — the hint is stored as a string-table identifier and translated on every read,
never at assignment. This is why changing language at run time updates every marker's tooltip.

## `CMapLocation` construction · `LoadSpot`

**Contract** — construction resolves the subject's authoritative record once — from the alife
simulation, permitting an absent one — and then builds the widgets from the spot type. A new
marker starts with its spot enabled, its pointer disabled and its hint enabled. Outside single
player the level name is fixed to the loaded level and never recomputed, because there is no
cross-level simulation to move the subject.

`LoadSpot` reads one named entry of the map-spot description data and builds up to three
independent presentations from it — a level-map spot, a minimap spot and a *complex* spot (a
richer level-map widget with side icons and a countdown). Each presentation may declare a
spot, a pointer, and the names of two border decorations. A presentation the description omits
is torn down; one it declares with an empty widget name is torn down too. **When all three end
up absent, the marker disables its own spot**, which is how a spot type can exist purely to
carry a hint or a serialization identity.

Four attributes on the entry set flags: a store marker makes it serializable, an offline
marker hides it while its subject is offline, a time-to-live arms expiry, and a
position-tracks-actor marker makes the marker follow the player rather than its nominal
subject.

**Invariant** — `LoadSpot` is **re-entrant against an existing marker**: it reuses a widget
that is already the right kind and only creates or destroys where the description differs.
That matters because the relation marker calls it again every time the player's standing with
its subject changes, and rebuilding all twelve widgets per change would be visible.

**Notes** — a missing entry in the spot description data is a hard failure naming the entry.
This is data every spot type must have, and a silently missing marker is far harder to
diagnose than a refusal at load.

## `Update`

**Contract** — refreshes the frame cache and reports whether the marker is still meaningful.
Idempotent within a frame. The order of the tests is the contract:

```text
FUNCTION update() -> actual
  IF already updated this frame THEN RETURN cached actuality

  IF this marker expires AND the expiry time has passed THEN
    actual = false ; RETURN                        # expiry wins over everything

  live = the subject, looked up among live objects
  IF the subject has an authoritative record
     OR (multiplayer AND the subject is live) THEN
    actual = true
    IF single player THEN recompute the level name
    recompute the position
  ELSE
    actual = false
```

**Invariant** — in single player the *authoritative record* is what makes a marker
meaningful, not the live object: a marker follows an entity across levels and while it is
offline, and the record is the only thing that exists in both cases. In multiplayer there is
no such record, so the live object stands in.

## `CalcPosition`

**Contract** — finds the subject's world position for this frame, from the best source
available. A marker flagged to track the actor takes the player's position and ignores its
nominal subject entirely — that is how "you are here" and directional prompts are built out of
the same machinery. Otherwise the live object's position is preferred; failing that, the
authoritative record's *drawable* level position, which is the record's own answer to "where
would you be if you were on screen".

**Invariant** — when neither source exists the previous position is **left in place** rather
than cleared. A marker whose subject briefly disappears — mid-transition — keeps its last
known position instead of jumping to the origin.

## `CalcDirection`

**Contract** — the direction the marker's arrow points, as a flat vector. The subject that is
*also the current view* uses the camera's direction, so the player's own arrow follows where
he is looking rather than where his body faces. Any other live subject uses its facing. A
subject that is not live points nowhere.

A marker that tracks the actor overrides all of that: its direction becomes the direction
*from the camera to the subject*, normalized — turning the marker into a bearing indicator
rather than a facing indicator.

## `CalcLevelName`

**Contract** — the name of the level the subject is on, derived from its authoritative
record's game-graph vertex. Recomputed **only when that vertex has changed** since the last
call, because the name lookup walks the game graph header. A marker with no record falls back
to the loaded level.

## `UpdateSpot`

**Contract** — the placement routine, run per marker per map per frame. It has two entirely
different jobs depending on whether the marker's subject is on the map being drawn.

**The subject is on this map.** Several refusals first, then placement:

```text
IF the alife simulation is running THEN
  RETURN IF this marker hides offline subjects AND the subject is offline
  RETURN IF the subject's record says it is not visible on the map

IF single player AND this marker belongs to a game task THEN
  show its border only while that task is the ACTIVE storyline or side task

map_pos = map.convert_world_to_local(position)
place the spot there
IF the spot's rectangle is visible on the map THEN
  IF the spot rotates with the map THEN set its heading to map heading + subject heading
  attach it to the map
IF single player THEN place and attach the border widget, if any
IF a pointer is wanted and the map says the spot is off its edge THEN place the pointer
```

**Invariant** — the conversion is performed **twice** when the map is a rotating one. The
first conversion is done in unrotated space, because visibility must be decided against the
map's axis-aligned bounds; the second, in rotated space, is the position actually drawn. A
rebuild that converts once gets either the wrong visibility or the wrong placement.

**The subject is on a different level.** The marker cannot be placed, so instead it points the
player toward the *way out*: a search over the game graph finds a route from the player to the
subject's coarse position, the route is walked backward to the last vertex still on the loaded
level, and a pointer is placed there — but only if that vertex is more than 45 metres away, so
that a player already standing at the exit is not shown a pointer at his own feet.

**Notes** — this off-level branch is the least healthy code in the file. The intended
behaviour, still present as commented-out code, was to point at the *level changer* on the
route; that search is disabled and the fallback is the route's last on-level vertex, which is
near the level changer but not on it. The diagnostic path for the disabled search is gated
behind a constant that is always false, so it is dead. A rebuild should implement the intent —
find the level changer whose graph vertex is on the route — and drop the fallback. The 45-metre
threshold is a tuned suppression radius with no derivation.

The route search uses the **actor's own terrain preference** as its cost model, so the route
shown is one the player could actually walk.

## `UpdateSpotPointer`

**Contract** — places the edge-of-map pointer for a marker whose spot is outside the visible
area. Asks the map where a pointer aiming at the marker's position should sit and which way it
should face; attaches it there. A pointer already attached this frame is left alone.

In single player it additionally computes the distance from the player to the marker, reports
it to the map when the marker belongs to a task, and **fades the pointer by distance** — fully
opaque under ten metres, then three coarser steps down to roughly forty per cent beyond a
hundred. The four bands are a presentation choice with no derivation; their effect is that a
screen full of distant pointers does not compete with a near one.

## `UpdateMiniMap` · `UpdateLevelMap`

**Contract** — place this marker on one map. Both do nothing when the spot is disabled or the
marker has no widget for that map. The level map prefers the **complex** spot when the marker
has one and falls back to the plain level spot; the two are mutually exclusive presentations
of the same marker, not layers.

## `GetSpotPointer` · `GetSpotBorder`

**Contract** — find the pointer and the border decoration belonging to one of the three
spots. The pointer is nothing when pointers are disabled for this marker. The border depends
on the marker's *pointer* state: an enabled pointer selects the "active" border and a disabled
one the "inactive" border, which is how the map shows at a glance which markers are being
tracked.

Borders are built **lazily**, on first request, from the names recorded at load. Most markers
never show one, and a border is a full widget; building twelve of them per marker up front
would dominate the cost of a map with a hundred markers.

**Notes** — an inactive border whose name is empty is simply never created, so a spot type can
declare an active border and no inactive one. The active border, by contrast, has a default
name per map kind, so every spot type has one whether it asked or not.

## `InitUserSpot`

**Contract** — pins a marker to a fixed world position on a named level rather than to an
entity. This is the player-placed map marker. Beyond storing the position, it finds the game
graph vertex nearest to it *on that level*, by scanning every vertex of the graph.

**Invariant** — the search starts from a squared-distance bound of 128, not from infinity.
That is a **hard cut-off**, not an optimization: a position more than about eleven metres from
any graph vertex finds nothing and the marker fails hard. The intent is that a player cannot
place a marker somewhere the simulation has no concept of. The original's own comment shows
infinity was the alternative considered. Note the bound is compared against a *squared*
distance while being written as a plain number, so the effective radius is its square root;
whether that was intended is not recoverable.

**Notes** — the scan is over every vertex of the whole game graph, filtered by level. It runs
once per placed marker, from a user action, so the cost is invisible; a rebuild with a
per-level vertex index does better.

## `SetHint` · `GetHint`

**Contract** — set and read the tooltip. Setting the literal identifier `disable_hint`
**turns hints off** for this marker and clears the text, rather than storing that string. That
sentinel is how the spot description data expresses "no tooltip" in a field that must contain
something. Reading returns nothing when hints are off, and otherwise translates through the
string table on every call.

## `HighlightSpot`

**Contract** — tints the level-map spot a given colour, or restores it to white. Restoring
checks the current colour first and writes only if it differs, because the write invalidates
the widget's cached geometry.

**Notes** — only the *level-map* spot is highlighted; the minimap and complex spots are not.
That asymmetry is not justified anywhere.

## `save` · `load`

**Contract** — writes the hint, the flag word and the owning task identifier, in that order.
Everything else is reconstructed: the widgets from the spot type (which the key holds, not the
marker), the position from the subject, the cache from scratch.

**Invariant** — the flag word is written and read as a single opaque value, so the **bit
positions are frozen** by the save format. A rebuild may not reorder the nine flags.

## `SpotSize`

**Contract** — the level-map spot's size, for callers laying out around it. Assumes that spot
exists.

## `CRelationMapLocation`

**Contract** — a marker whose appearance and visibility are a function of the *relation*
between its subject and the player. Used for every character marker on the minimap. Its update
runs after the base update and does four things.

**Resolve the relation.** Preferably between the two authoritative records — which works for a
subject that is offline or on another level — otherwise between the two live objects. Either
resolution failing makes the marker non-actual. A creature record additionally supplies
whether the subject is alive.

**Choose the spot type from the relation.** A dead subject gets the corpse spot type; a live
one gets the spot type the relation registry names for that standing. **When the chosen type
differs from the current one, the widgets are rebuilt** — this is the re-entrant `LoadSpot`
call, and it is why a character's dot changes colour the moment he turns hostile.

**Decide visibility**, which is the mechanic that matters:

- A **friendly or neutral** subject is always visible. You always know where your own faction
  is.
- An **enemy** is visible only while the player can *actually see him*, resolved through the
  player's own vision memory — unless he is dead, or unless the player wears a helmet with an
  enemy-detection range and the enemy is inside it. That helmet field is the entire mechanical
  value of those helmets.
- A **corpse** is visible only when it is within three metres of the player *vertically*. The
  vertical-only test is deliberate — the horizontal distance test beside it is commented out —
  and its effect is that corpses on your own floor show and corpses a storey above or below do
  not. A rebuild reproducing the commented-out horizontal test changes how looting reads.

**Suppress duplicates.** When this marker is visible, it asks the manager for every other
marker on the same subject and hides its own minimap or level-map presentation wherever
another marker already provides one. A character who is also a quest target shows the quest
marker, not two overlapping dots.

**Notes** — a marker that transitions from hidden to visible resets its minimap spot's
transform animation, so the dot plays its appearance animation from the start rather than
mid-way.

The visibility computation queries the player's vision memory once per enemy marker per frame.
With many characters on screen this is the marker system's dominant cost.
