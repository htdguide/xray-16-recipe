# src/xrUICore/Buttons/UIBtnHint.cpp

> A hint box built from one of two XML layouts depending on which game's data is mounted, claimed by one widget at a time and drawn last.

**Needs** — [`UIBtnHint.h`](UIBtnHint.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Windows/UIFrameWindow.h`](../Windows/UIFrameWindow.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`XML/xrUIXmlParser.h`](../XML/xrUIXmlParser.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIBtnHint.h`](UIBtnHint.h.md)
**Tier floor** — T3.

## Purpose

The hint box is the one widget whose construction must cope with two incompatible shipped
layouts, because the three games describe it differently. This file detects which and builds
accordingly, then provides claim/release and a deferred draw.

## State

```text
RECORD ButtonHint EXTENDS FrameWindow
  owner            : optional<Window>   # who claimed it; none == free
  text             : Static             # the hint's text child
  border           : optional<FrameLine># present only in the older layout
  wanted_this_frame: bool
```

**Invariants** — `border` present and the frame window's own nine-slice texture absent are the
two mutually exclusive layouts; every method that colours or resizes the box branches on
which.

## Construction

**Contract** — loads the hint layout document, probes for a texture attribute under the hint
element, and picks a layout from the answer: if the element carries a texture, the box itself
is a nine-slice frame window and there is no separate border; if not, the box is a plain
window and a stretched-line border child is created and initialized from a sibling element.
The text child is created and initialized in both cases. Runs once per instance, at startup.

**Notes** — the probe is the only feature detection of its kind in the toolkit: everywhere
else the games' differences are absorbed by making an attribute optional. Here the two
layouts need a different child tree, so the shape of the data selects the shape of the
widget. The probed path is the newer game's; the fallback is the older game's.

## `set_hint_text`

**Contract** — claims the hint for the given widget, sets the text from a localization key,
and resizes the box around it. The two layouts resize in different axes: the bordered layout
fits the *width* to the text with a 30-unit margin and a floor of 80 units, leaving the height
fixed; the frame layout fits the *height* to the wrapped text plus 20 units, leaving the width
fixed. Finally the text's colour animation is restarted so the hint fades in from its
beginning each time it appears.

**Notes** — the border's width is set but the stretched-line widget ignores width changes, so
in the bordered layout the border does not actually track the text. The original marks this
as a known defect. Reproduce the intent — the border should track — rather than the bug.

## `on_render`

**Contract** — if the hint was wanted this frame: update the text child (so its colour
animation advances), recolour the box — the frame or the border, whichever exists — to opaque
white at the text's current alpha so the box fades with the text, draw the whole box, and
clear the wanted flag. Otherwise do nothing.

**Invariants** — the wanted flag is set by whoever is drawing a widget that owns the hint and
cleared here, so a hint disappears the frame after its owner stops drawing, without any
explicit release.

## `draw_` / `owner` / `discard`

**Contract** — `draw_` sets the wanted flag; `owner` reports the current claimant; `discard`
releases the claim unconditionally. Release is the claimant's responsibility and happens when
the pointer leaves it.
