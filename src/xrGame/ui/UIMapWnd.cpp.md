# src/xrGame/ui/UIMapWnd.cpp

> The map screen: it places every level map on one world map, pans and zooms that world map,
> and delegates *how the view gets from here to there* to a goal-driven planner rather than to
> a tween.

**Needs** — [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMap.h`](UIMap.h.md) · [`UIMapWndActions.h`](UIMapWndActions.h.md) · [`UIMapWndActionsSpace.h`](UIMapWndActionsSpace.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`map_manager.h`](../map_manager.h.md) · [`map_location.h`](../map_location.h.md) · [`map_spot.h`](../map_spot.h.md) · [`map_hint.h`](map_hint.h.md) · [`GametaskManager.h`](../GametaskManager.h.md) · [`GameTask.h`](../GameTask.h.md) · [`Actor.h`](../Actor.h.md) · [`xrUICore/ScrollBar/UIFixedScrollBar.h`](../../xrUICore/ScrollBar/UIFixedScrollBar.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [`xrUICore/Hint/UIHint.h`](../../xrUICore/Hint/UIHint.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIMapWnd.h`](UIMapWnd.h.md)
**Tier floor** — T3.

## Purpose

The screen behind the PDA's map page and the task page's map panel. Its two genuinely
interesting decisions are:

1. **There is one map, not many.** The world map is the only thing that is panned or zoomed;
   every level map is a child of it that recomputes its own rectangle from the world map's
   transform. Zooming the world map zooms everything for free.
2. **View transitions are planned, not tweened.** Asking the screen to show a place sets a
   *goal* — a target map and a target centre — and a planner decides whether that requires
   zooming out first, moving, resizing, or nothing, and drives it over time. See
   [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md).

The screen is instantiated at most once and publishes itself through a process-wide handle,
which the source itself labels temporary.

## State

```text
RECORD MapScreen EXTENDS Window
  global_map      : GlobalMap             # the only pannable, zoomable surface
  game_maps       : map<text, LevelMap>   # by lowercased level name; insertion-ordered
  main_frame      : FrameWindow           # decoration
  level_frame     : Window                # THE VISIBLE AREA; its absolute rect is the clip
  map_header      : optional<FrameLine>
  scroll_h, scroll_v : optional<ScrollBar>
  scroll_mode     : bool                  # whether this document wants scroll bars at all

  action_planner  : MapActionPlanner
  tgt_map         : CustomMap             # the planner's goal
  tgt_center      : Point                 # ... in identity world-map units
  current_zoom    : real
  view_actor      : bool                  # centre on the actor at the next show
  prev_actor_pos  : Point

  nav_buttons     : list<optional<Button>>  # nine, indexed by position
  nav_parent      : optional<Static>
  map_move_step   : real                  # canvas units per key repeat; from the document
  location_hint   : MapLocationHint       # ONE hint, shared; drawn out of band
  properties_box  : PropertiesBox         # the marker context menu
  cur_location    : optional<MapLocation>  # what the context menu is about
```

**Invariants**

- **The visible area is the level frame's absolute rectangle**, and it is pushed into every
  map's working area at every show and every update. Nothing else defines what "on screen"
  means for a map, a marker, or a hint.
- Level map keys are lowercased. The registry is ordered, and a level's *index* is its
  position in that order — which is what the rest of the game uses to name a map compactly.
  A rebuild must keep the ordering stable across a session; it need not match the original's.
- Two level maps' placement rectangles must not overlap, and every one must lie inside the
  world map's bounds. Both are checked only in a development build, so bad data ships silently.
- The world map is *emptied and refilled* at every show: all level maps are detached, then
  re-attached and given the current working area. A screen that is shown while already shown
  therefore rebuilds cleanly.
- Exactly one hint exists for the whole screen, owned by whichever widget last claimed it.
  Precedence is by *location level*, not by recency — see the hint section.

## `Init`

**Contract** — load the named layout document; decline (without failing) when it is absent and
the caller said the document is optional. Apply the screen's own section, read the pan step,
then build the main frame, the visible-area frame and the header — each with a **fallback
element path**, because the shipped games nest them differently. Optionally build two scroll
bars. Build the navigation cluster, the hint, the world map, and one level map per entry of
the level-map section for the current game kind. Register everything for directional
navigation, set up the planner, and create the hidden context menu.

```text
FUNCTION init(document_name, root_path, required) -> bool
  doc <- load_layout(document_name);  IF absent AND NOT required THEN RETURN false
  apply(doc, root_path + ":main_wnd", self)
  map_move_step <- doc.attribute(root_path, "map_move_step") OR 10

  main_frame  <- first of [root_path + ":main_map_frame",
                           root_path + ":main_wnd:main_map_frame"] that exists
  level_frame <- first of [root_path + ":level_frame",
                           root_path + ":main_wnd:main_map_frame:level_frame"]
  header      <- first of [root_path + "main_map_header",
                           root_path + ":main_wnd:map_header_frame_line"]   # optional

  IF doc says scrolling is enabled, OR this is the first game THEN build_scroll_bars(doc)
  build_nav_cluster(doc, root_path)
  location_hint <- from doc at root_path + ":map_hint_item", drawn by the screen itself

  global_map <- new GlobalMap(self);  global_map.initialize()
  level_frame.adopt(global_map)
  global_map.optimal_fit(level_frame.rect)
  global_map.min_zoom <- its resulting zoom      # "fully zoomed out" means "fits the frame"
  current_zoom <- that

  section <- single player ? "level_maps_single" : "level_maps_mp"
  FOR EACH level_name IN config[section]                 # the key set, values unused
    key <- lowercase(level_name)
    FAIL WITH duplicate IF game_maps HAS key
    game_maps[key] <- new LevelMap(self) initialised from that level, fitted to the frame

  action_planner.setup(self);  view_actor <- true
  properties_box <- a hidden 300x300 context menu named "property_box"
  RETURN true
```

**Notes**

- **The minimum zoom is discovered, not configured**: it is whatever zoom makes the world map
  exactly fit the visible frame. That is why "zoom reset" and "show the world" are the same
  action, and why the minimum differs between screen layouts.
- The fallback element paths exist because three games' documents place the same widgets at
  different depths. A rebuild with one layout needs one path.
- The level-map section is used as a *set of names*; the values are ignored. Each named level
  must also have a configuration section of its own, or construction fails loudly — the map's
  geometry lives there.
- The scroll bars try the fixed-thumb variant first and fall back to the proportional one,
  because the fixed variant needs texture metrics a style may not ship. Their step is a tenth
  of the visible frame and their page is the whole of it.

## `Show`

**Contract** — on either transition, detach every map's markers and hide the world map. When
showing: show the world map, give it the current visible area, re-attach and show every level
map with the same visible area, and — the first time only — centre on the actor. Also notify
the actor that the local map was opened, which scripts listen for. Always drop the hint.

**Notes** — the "notify the actor" call is an *information packet*, the game's one-way channel
from engine to script; opening the map is a scriptable event. This is the pattern the chapter
context calls out: a UI action becomes a game action through an event, never through a direct
call into game logic.

## `Activated`

**Contract** — when the screen becomes active, re-centre on the actor if the actor has moved
more than three units since the last centring. Otherwise leave the view where the player left
it.

**Notes** — the three-unit dead zone is what makes the map remember where you were looking
between openings, while still following you across a level. It is a constant with no
configuration entry.

## `SetTargetMap` — the only way to ask for a view

**Contract** — four forms resolving to one: record the target map, compute the target centre
in **identity world-map units** (that is, at zoom one), optionally request the maximum zoom,
and reset the planner so it re-plans. Targeting the world map itself instead forces the
minimum zoom and centres on the current visible middle. A name that is not a registered level
is ignored with a log line.

```text
FUNCTION set_target_map(map, world_pos, zoom_in)
  tgt_map <- map
  IF map IS global_map THEN
    set_zoom(global_map.min_zoom)
    tgt_center <- (centre of visible area - global_map.absolute_position) / current zoom
  ELSE
    IF zoom_in THEN set_zoom(global_map.max_zoom)
    tgt_center <- (map.world_to_local(world_pos, for_drawing) + map.position) / current zoom
  reset_planner()
```

**Notes** — dividing by the current zoom is what makes the target *zoom-independent*: the
planner will later multiply it by whatever zoom it is animating toward. Getting this wrong
produces a view that overshoots proportionally to the zoom change, which is the characteristic
failure of a naive rebuild.

## `ViewActor`, `ViewGlobalMap`, `ViewZoomIn`, `ViewZoomOut`

**Contract** — centring on the actor targets the level map for the currently loaded level at
the actor's horizontal position, with a zoom-in, and remembers that position; if the loaded
level has no map, it targets the world map instead. The other three reset to the world map or
step the zoom. All four are inert while the world map is locked — that is, while the planner
is mid-animation.

## `UpdateZoom`

**Contract** — multiply or divide the zoom by 1.2 and clamp it to the world map's range. If
the clamp swallowed the change, report that nothing happened. Otherwise set the target centre
to the *current* visible middle — so a zoom keeps the middle of the screen fixed — re-plan,
and drop the hint.

**Notes** — the return value is inverted relative to its name: it reports *true* when the zoom
did **not** change. Only the planner reads it.

## Input

**Contract** — key presses map bound actions to reset-to-world, centre-on-actor and toggle the
legend; key *holds* map to zoom in, zoom out and the four pan directions, each pan moving by
the document's step. A controller stick maps to a normalised pan in any direction. The pointer
pans while its primary button is held, zooms on the wheel, and opens the marker context menu
on a secondary-button release — but only while the cursor is inside the visible area and the
world map is unlocked.

**Notes** — pan and zoom are on *hold*, not on press, so they repeat; the view actions are on
press, so they fire once. The wheel's two directions are deliberately crossed relative to the
usual convention: wheel-down zooms *in*. That matches the shipped game and is the kind of
thing a rebuild reverses by accident.

## The hint, and its precedence rule

**Contract** — one hint serves the whole screen. Setting it from a plain string succeeds only
when no owner holds it. Setting it from a marker succeeds when nothing holds it, **or when the
new marker's location level is higher than the current owner's** — so a more important marker
displaces a less important one under the same cursor. Setting it from a task always succeeds
and is allowed to escape the map's visible area into the whole canvas. Any hint that does not
fit its permitted rectangle is dropped rather than clipped. A widget may only hide the hint if
it owns it.

**Notes** — the hint is drawn out of band by the screen's owner, not in the child traversal,
so it lands above everything including the map's children. The task form's wider rectangle is
the whole virtual canvas, which is how a task hint can overhang the PDA frame.

## `ActivatePropertiesBox` — the marker context menu

**Contract** — clear the menu, and give up unless the clicked widget is a marker with a
location behind it. Ask a script hook to add whatever entries it wants, passing the location's
object, level and hint text. For a *player-placed* marker additionally offer rename and delete.
If anything was added, size the menu to its contents and open it at the cursor, clamped inside
the screen.

**Notes** — the entry set is script-extensible by design: the engine contributes two entries
and only for user markers, and everything else on that menu in the shipped games comes from
Lua. The menu's selection is likewise handed back to a script hook. This is the second place
the chapter's "a UI action becomes a game action through an indirection" rule shows up
concretely — here the indirection is a named script function rather than a notification.

## `SpotSelected`

**Contract** — clicking a marker that belongs to a game task makes that task the active one.
A marker with no task behind it does nothing.

## Scrolling

**Contract** — when the document asked for scroll bars, the bars' ranges are the world map's
current pixel size and their positions are the negated world-map origin, refreshed after every
pan and after every planner step. Dragging a bar sets the corresponding world-map coordinate
directly.

**Notes** — the ranges are recomputed from the map's size on every refresh because the map's
size *is* the zoom. There is a transposition in the horizontal bar's range — it takes its
minimum from the vertical bar — which is harmless only because both minima are the same.

## `Update` and `Draw`

**Contract** — the update refreshes the world map's working area from the visible frame,
updates the child tree, steps the planner and polls the held navigation buttons. The draw
draws the child tree and then the navigation cluster separately, because the cluster is
parented to a widget that is deliberately outside the normal order.

## `MapLocationRelcase`

**Contract** — when a map location is about to be destroyed, release the hint if it is owned
by a marker for that location. The hint's owner may also be a task list row, which holds no
location, so the check is conditional.

**Notes** — this is the whole of the screen's participation in object teardown, and it is
necessary because the hint holds a raw reference to a widget whose lifetime the map manager
controls. A rebuild with owning references still needs the *decision*: a hint about something
that no longer exists must be dismissed, not left stale.
