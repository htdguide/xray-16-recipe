# src/xrGame/ui/UILabel.cpp

> Nothing: the whole file is a disabled implementation of a framed text label that animated its colour from a light-animation curve.

**Needs** — _(none: the file defines nothing)_
**Used by** — reached through its declarations in [`UILabel.h`](UILabel.h.md); callers name that, not this file.
**Tier floor** — T4: the file contributes no code.

## Purpose

Commented out in its entirety, as is its header. It preserved an early framed-label widget —
a stretchable frame line with a text layer drawn over it, whose colour was driven each frame
from a named light-animation curve resolved at construction.

Every part of it survives elsewhere and better: chapter 15's frame-line window carries text
natively, and colour animation on a widget is a general facility applied by name rather than a
property of one widget type. A rebuild drops the file.

**Notes** — one detail is worth keeping, because the surviving mechanism inherited it: the
animation's clock was started **lazily at the first update**, not at construction, so a label
built during a load screen did not appear to be several seconds into its animation when it
was first shown. The same lazy start appears in the colour-animation wrapper that replaced it.
