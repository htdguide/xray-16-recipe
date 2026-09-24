# src/xrUICore/arrow — the needle

> A static whose rotation tracks a normalized value, and which swings toward a new value at a
> bounded angular speed instead of snapping to it.

Part of [chapter 15](../README.md).

## What this directory is responsible for

One widget: the analogue gauge needle. The radiation detector, the compass-like readouts and
the dial faces in the shipped heads-up displays are all this. It maps a value in a range onto
an angle in a range, and it animates.

## The load-bearing ideas

**The needle has inertia, and the inertia is the point.** A detector needle that jumped to its
new reading would be unreadable and would not look like an instrument. The widget moves toward
its target at a bounded angular speed per frame, so a large change takes visible time and a
small one is immediate.

**Rotation is a property of the quad, not of the widget tree.** The needle rotates its own
drawable about an authored pivot; nothing in the tree knows a widget can be rotated. The
aspect-correction factor applied during rotation is what keeps the needle from shearing on a
wide display.

**Two ranges, both authored.** A value range and an angle range come from the layout, and the
map between them is linear. That is the whole configuration.

## The twins

| Twin | Role |
|---|---|
| [`ui_arrow.cpp`](ui_arrow.cpp.md) | The value-to-angle map, and the bounded-speed chase toward a new value |
| [`ui_arrow.h`](ui_arrow.h.md) | Its declaration |
