# src/xrGame/ui/UICarPanel.h

> An empty placeholder where a vehicle dashboard screen used to be.

**Needs** — _(none)_
**Used by** — [`UICarPanel.cpp`](UICarPanel.cpp.md)
**Tier floor** — T4: no content

## Purpose

The file declares nothing. The car dashboard it once held — speed, fuel, damage readouts for
a driven vehicle — was removed, and the empty file survives only so the build description and
the sibling implementation file still resolve. A rebuild deletes it and the vehicle panel is
one of the features it does not owe.
