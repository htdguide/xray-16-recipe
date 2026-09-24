# src/xrGame/ui/ArtefactDetectorUI.h

> The four display behaviours an artefact detector can have, from a blinking lamp to a
> world-space map of nearby artefacts drawn on the device's own screen.

**Needs** — [`ArtefactDetectorUI.cpp`](ArtefactDetectorUI.cpp.md) · [`../../xrUICore/Windows/UIFrameLineWnd.h`](../../xrUICore/Windows/UIFrameLineWnd.h.md) · [`../../xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`AdvancedDetector.cpp`](../AdvancedDetector.cpp.md) · [`CustomDetector.cpp`](../CustomDetector.cpp.md) · [`EliteDetector.cpp`](../EliteDetector.cpp.md) · [`SimpleDetector.cpp`](../SimpleDetector.cpp.md) · [`ArtefactDetectorUI.cpp`](ArtefactDetectorUI.cpp.md)
**Tier floor** — T1: the elite and advanced displays write bone transforms and light
parameters that the renderer reads by layout each frame

## Purpose

An artefact detector is a held item whose *model* is its display: the readout is not a screen
overlay but geometry and lights on the object in the player's hands. This header declares the
one interface every detector display satisfies and the four that ship, each keyed to a
detector tier that the game data names.

The header is substantive even though only one of the four classes is implemented in the
sibling file: the others are implemented beside their detector items, so what this header
demands of an implementor is the only place the family is described as a family.

## State

```text
RECORD DetectorDisplay                # the interface
  # no state of its own; each filling owns the parts of the held model it drives

RECORD DetectorWave                   # the sweep bar on the simple detector
  velocity : real                     # canvas units per second, commanded by the item
  step     : real                     # the wrap period, from the layout document

RECORD SimpleDisplay
  parent            : detector item
  flash_bone        : bone handle     # the lamp that pulses
  on_off_bone       : bone handle     # the power lamp
  flash_until       : time            # when the current pulse ends
  flash_light       : light
  power_light       : light
  flash_animation   : colour animation
  power_animation   : colour animation

RECORD EliteDisplay
  work_area     : widget tree drawn in world space on the device's screen
  palette       : map<text, widget>   # one blip prototype per artefact class name
  to_draw       : list<(widget, world position)>   # rebuilt every frame
  attach_offset : transform           # device screen plane relative to the attach bone

RECORD AdvancedDisplay
  target_direction : vector           # where the needle wants to point
  current_rotation : real             # where it points now
  angular_speed    : real             # bounded; the needle chases, it does not snap
  needle_bone      : bone handle
```

Invariant across all four: the display owns no simulation state. Which artefacts are near, and
how near, is decided by the detector item; the display only renders that decision.

## `CUIArtefactDetectorBase`

**Contract** — The interface. One operation, `update`, called once per frame by the owning
detector while it is in the player's hands; the default does nothing, which is what a detector
with no display uses. Destruction must release whatever renderer resources the filling took —
lights, bone callbacks — in a defined order, because the model may be unloaded immediately
afterwards.

## `CUIDetectorWave`

**Contract** — A stretchable frame line that scrolls horizontally at a commanded speed and
wraps, giving the simple detector its sweeping bar. Configured from a layout document that
supplies the frame-line geometry plus the wrap period. Algorithm in
[`ArtefactDetectorUI.cpp`](ArtefactDetectorUI.cpp.md).

- `SetVelocity(v)` — the item sets this from artefact proximity; the bar sweeps faster as an
  artefact gets closer.

## `CUIArtefactDetectorSimple`

**Contract** — The lamp-only display. Two named bones on the detector model carry two lights;
`Flash(on, relative_power)` pulses the first for a bounded time at an intensity scaled by
proximity, and the second shows power state. The colour of each pulse comes from a named
colour animation rather than a constant, so the lamp's look is authored in data.

**Invariants** — The pulse has a deadline: a flash started and never cleared must extinguish
itself on its own, because the item that started it may be holstered mid-pulse.

## `CUIArtefactDetectorElite`

**Contract** — The top-tier display: a small map of nearby artefacts drawn *in world space* on
the plane of the detector's screen. The detector item calls `Clear` then
`RegisterItemToDraw(position, palette name)` once per artefact it senses, each frame; `Draw`
then places one blip widget per registration by projecting the world position through the
screen plane's transform.

**Invariants** — The registration list is rebuilt every frame and must be cleared first;
blips are *not* retained between frames. The palette is a fixed map from artefact class name
to blip appearance, loaded once — an artefact whose name is not in the palette cannot be
drawn, which is how the data decides what this detector is able to show.

**Notes** — This is the one place in the whole UI chapter where widgets are drawn into the
world rather than onto the canvas. The toolkit's half-pixel texel correction is suppressed for
this case (see chapter 15); a rebuild that forgets that gets a visibly blurred device screen.

## `CUIArtefactDetectorAdv`

**Contract** — The needle display: one bone on the model rotates to point at the strongest
sensed artefact. `SetValue(power, direction)` records the target; `update` advances the
current rotation toward it at a **bounded angular speed**, so the needle sweeps rather than
teleports. The rotation is applied by installing a callback on the needle bone that the
animation system invokes while composing the pose, and `ResetBoneCallbacks` must remove it
before the model goes away.

**Invariants** — The bounded speed is the whole point: an unbounded needle is unreadable, and
the bound is what makes the device feel mechanical. The current rotation must persist across
frames — it is the integrator — and must be reset when the item is re-equipped, or the needle
starts a sweep from wherever it was left.
