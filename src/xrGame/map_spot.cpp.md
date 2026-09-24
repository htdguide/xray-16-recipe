# src/xrGame/map_spot.cpp

> The widgets a map marker is drawn as: the clickable icon, the edge pointer, the minimap dot that changes with height, and the rich spot with satellite icons and a countdown.

**Needs** — [`map_spot.h`](map_spot.h.md) · [`map_location.h`](map_location.h.md) · [`ui/UIMapWnd.h`](ui/UIMapWnd.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`ui/UIHelper.h`](ui/UIHelper.h.md) · [`ui/UIInventoryUtilities.h`](ui/UIInventoryUtilities.h.md) · [`GameTask.h`](GameTask.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`Level.h`](Level.h.md) · [`xrUICore/XML/UITextureMaster.h`](../xrUICore/XML/UITextureMaster.h.md) · [`Include/xrRender/UIShader.h`](../Include/xrRender/UIShader.h.md)
**Used by** — [`map_spot.h`](map_spot.h.md)
**Tier floor** — T3: widget behaviour driven by description data

## Purpose

A map marker decides *where* it is; these widgets decide *what it looks like and what happens
when you touch it*. Four kinds, all built from the same description data, differing in three
ways that each encode a real decision: whether it can be clicked, whether its picture depends
on height, and whether it carries satellites that must scale with it.

## State

```text
RECORD MapSpot EXTENDS Picture
  marker        : MapLocation      # borrowed; the marker this draws
  scales        : bool             # does this spot grow with map zoom
  scale_bounds  : (real, real)     # min and max zoom at which it is drawn
  detail_level  : int              # which map detail level it appears at
  border        : optional<Picture>  # the active-task ring
  origin_size   : (real, real)     # the authored size, for proportional rescaling
  focused       : bool

RECORD MiniMapSpot EXTENDS MapSpot
  above, normal, below : (shader, texture rectangle)   # three height variants

RECORD ComplexMapSpot EXTENDS MapSpot
  left, right, top : Picture       # satellite icons
  timer            : Picture with text
  timer_finish     : int           # game time, milliseconds; 0 with infinite set means none
  infinite         : bool
  tick_accumulator : int           # milliseconds since the countdown text was last rewritten
```

**Invariant** — a spot borrows its marker and never owns it. The marker outlives its widgets;
teardown is always marker-first.

**Invariant** — the authored size is captured once at load and every later rescale is computed
from *it*, never from the current size. Scaling from the current size compounds rounding on
every zoom step and the satellites drift away from the spot within a few zooms.

## `CMapSpot::Load`

**Contract** — builds the widget from one named entry of the description document: the picture
itself, whether it scales and between which bounds, its detail level, and an optional border
child which starts hidden.

**A spot that does not rotate with the map is width-corrected for the display's aspect ratio
and set to stretch its texture; a rotating one is not.** That asymmetry is the load-bearing
part. A rotating spot is drawn through the map's own rotation, which already carries the
aspect correction; applying it twice would make a rotated icon oval. A non-rotating spot is
drawn in screen space and must correct for itself.

**Invariant** — a spot that declares itself scalable must declare both bounds, except in
*Shadow of Chernobyl* mode where the data predates them. That exception is a compatibility
carve-out for the oldest of the three games' data, not a design choice.

## `CMapSpot::Update` · `OnFocusLost`

**Contract** — while the cursor rests on a spot, a tooltip is requested after half a second;
losing focus withdraws it. The half second is **scaled by the game's time factor**, so the
delay is measured in game time rather than real time — a consequence of using the game clock
for a user-interface delay, and visibly wrong while time is accelerated. A rebuild should use
real time here.

## `CMapSpot::OnMouseDown`

**Contract** — the click behaviour, and it is asymmetric by design. A **left** click selects
the spot **only if its marker belongs to a game task**; a spot that is not a task target
swallows nothing and the click falls through to the map beneath, which is how the player pans
the map by dragging over markers. A **right** click always sends the secondary selection,
which is what opens the marker's context action.

## `CMapSpot::GetHint` · `show_static_border` · `mark_focused`

**Contract** — the tooltip is the marker's, not the widget's — a widget never owns display
text. The border is shown or hidden on request, which is how the map indicates the active
task. Marking focused records that the spot was the one under the cursor this frame.

## `CMapSpotPointer`

**Contract** — the edge-of-map arrow. Identical to the base except that **it has no tooltip**:
a pointer stands for something that is off the map, so a tooltip at its position would
describe a place the player is not looking at.

## `CMiniMapSpot::Load` · `Draw`

**Contract** — loads up to three texture variants — above, normal and below — and at draw time
picks one by comparing the marker's last world height against the viewer's.

```text
FUNCTION draw()
  IF a view entity exists AND both the above and below variants were loaded THEN
    d = viewer.height - marker.height
    IF      d >  1.8 THEN use the BELOW variant     # the marker is beneath the player
    ELSE IF d < -1.8 THEN use the ABOVE variant
    ELSE                  use the NORMAL variant
  draw as a normal spot
```

**Invariant** — 1.8 metres is roughly the player's own height, and it is the threshold in both
directions. Its effect is that anything within one storey reads as "on my level". It is a
tuned constant; the choice of the player's height for it is the only derivation available.

**Invariant** — the variant switch requires **both** extra variants to have loaded. A spot
type declaring only one is drawn with its normal texture always, silently. That is a
reasonable failure mode — a half-configured spot still draws — and a rebuild should keep it
rather than failing.

**Notes** — loading the variants abuses the shared picture item as scratch: each texture is
initialized into it to discover its shader and rectangle, which are then copied out, and the
item's original rectangle is restored at the end. The restore is essential; without it the
spot draws with the last variant's rectangle. A rebuild loads each variant into its own slot
and the restore disappears.

A texture name containing the delimiter denotes a region of an atlas, and its rectangle may be
overridden by explicit coordinates; a plain name uses the whole texture's rectangle. That is
the description data's convention, shared with the rest of the user interface.

## `CComplexMapSpot::Load` · `CreateStaticOrig`

**Contract** — builds the rich spot: the base picture plus four children at fixed names —
left, right and top satellite icons and a countdown. Each child is created as a
*position-remembering* picture and attached, with the description document's local root
temporarily moved to this spot's node so the children can be addressed by bare name.

**Invariant** — the local root must be restored afterwards. The document is shared by every
marker in the game and a stranded root would make every subsequent load fail.

## `CComplexMapSpot::SetTimerFinish` · `Update`

**Contract** — set the countdown's end, in game time; a non-positive value means *no
countdown*, which hides the timer and marks it infinite. The update rewrites the countdown
text **at most about three times a second** — the accumulator threshold is 310 milliseconds —
because the text is drawn at minute resolution and rebuilding it every frame is pure cost. It
then shows or hides the timer according to whether the spot is enabled, whether the timer has
a meaningful width, and whether the end is still in the future.

**Notes** — the branch that fires when the countdown has *expired* is commented out; the
original intent was for an expired complex spot to disable itself. As it ships, an expired
countdown simply stops updating its text and the timer hides, while the spot remains. A
rebuild should decide deliberately rather than inherit the empty branch.

The width test — a timer narrower than five units is not shown — is a proxy for "this spot's
description did not configure a timer", since an unconfigured child has near-zero size. A
rebuild should record the absence explicitly.

## `CComplexMapSpot::SetWndSize`

**Contract** — resizing the spot rescales every position-remembering child **proportionally,
from its authored geometry**, by the ratio of the new width to the authored width; then the
countdown is re-centred horizontally under the spot. A spot with no recorded authored size
does nothing, which is the guard against rescaling before load has finished.

**Invariant** — the scale factor comes from width alone and is applied to both axes, so the
satellites keep their aspect even if the spot is resized non-uniformly. That is intended: a
map spot is square in the shipped data.

## `CUIStaticOrig`

**Contract** — a picture that records its authored position and size once and can be rescaled
from them any number of times without drift. `InitWndOrigin` captures; `ScaleOrigin`
multiplies both by a factor and applies the result.

**Invariant** — the capture must happen after the description data has been applied and before
the first rescale. That ordering is why the creation helper does both in one call rather than
leaving the capture to the caller.
