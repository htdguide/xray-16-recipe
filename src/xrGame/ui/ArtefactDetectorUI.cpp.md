# src/xrGame/ui/ArtefactDetectorUI.cpp

> The scrolling sweep bar of the simple artefact detector.

**Needs** — [`ArtefactDetectorUI.h`](ArtefactDetectorUI.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../../xrUICore/XML/xrUIXmlParser.h`](../../xrUICore/XML/xrUIXmlParser.h.md)
**Used by** — [`ArtefactDetectorUI.h`](ArtefactDetectorUI.h.md)
**Tier floor** — T3: one position integrator against frame time

## Purpose

Carries the one detector display whose behaviour is purely a widget's: the bar that sweeps
across the simple detector's face. The other three displays declared in
[`ArtefactDetectorUI.h`](ArtefactDetectorUI.h.md) drive model bones and lights and are
implemented beside their detector items, which is why this file is nearly empty. The split is
an accident of where each class's dependencies live; a rebuild may put all four together.

## State

See [`ArtefactDetectorUI.h`](ArtefactDetectorUI.h.md).

## `CUIDetectorWave::Update`

**Contract** — Advances the bar's horizontal position by the commanded velocity times the
frame's elapsed time, then wraps it back into a band one `step` wide. Called once per frame
while the detector is drawn. No allocation, no blocking.

```text
FUNCTION update()
  x = position.x + velocity * frame_delta

  # The band is [-2*step, 0]: the bar is authored wider than the visible
  # face and slid leftwards, so the wrap seam is always off-screen.
  IF x > 0             THEN x = x - step
  ELSE IF x < -2*step  THEN x = x + step

  position.x = x
```

**Notes** — Velocity may be negative, which is why both wrap arms exist rather than one
modulo: the item drives the bar in either direction depending on whether the player is closing
on an artefact.

The band bounds — one `step` above zero, two below — are not symmetric and are not tunable.
They exist because the bar's own width is authored to exceed the visible face by one `step`,
so the wrap always happens under the bezel. A rebuild must keep the same relationship between
the authored bar width and the wrap period or the seam becomes visible in the shipped data.

## `CUIDetectorWave::InitFromXML`

**Contract** — Configures the widget as a stretchable frame line from the named element of a
layout document, then reads one extra attribute, the wrap period. Fails the way every layout
read fails when the element is absent.
