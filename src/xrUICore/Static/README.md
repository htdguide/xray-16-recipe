# src/xrUICore/Static — the picture-and-text widget, and the quad under it

> Almost every control in the chapter is a static with something added. This is where a
> widget acquires a texture, a text block, a colour animation and a hover hint — and where,
> one layer down, a rectangle becomes triangles.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The **static** is the composite: a window that may carry a drawable quad, a text control, a
colour animation, a transform animation and a delayed tooltip, drawn in a fixed order. It is
the base of the button, the list row, the progress bar and most of what chapter 25 builds.
Its two sub-parts — the quad and the text — live in this directory and in
[`Lines/`](../Lines/README.md) respectively.

The **static item** is the quad: the single place in the chapter where a widget becomes
geometry. A rectangle, a sub-rectangle of an atlas page, a colour, a mirroring mode and an
optional rotation, clipped against the current scissor frustum and emitted as a triangle fan.

Two decorations sit beside them. An **animated static** plays a sprite sheet: a frame count
and a column count describe a grid on one texture, a duration spreads the frames over time,
and the texture sub-rectangle is stepped rather than the texture swapped. A **light-animation
controller** lets any widget be driven by a named curve from the engine's light-animation
library, reinterpreting that curve's colour channels as whatever the widget needs — text
colour, texture colour, either whole or alpha only.

## The load-bearing ideas

**Unset means derive, not zero.** A quad's size and texture rectangle each carry a validity
flag; if either is unset when the item is first drawn it is filled in from the material's base
texture resolution, so a picture with nothing said about it draws at its page's full size.
Re-texturing clears both flags so they re-derive.

**Texture coordinates are normalised at draw time**, from the page's actual reported
resolution, because the page may not be resident when the layout was read. That is what lets
one atlas description serve pages shipped at different sizes.

**No batching across widgets.** Each quad opens and flushes its own primitive batch. Draw
order is the widget tree's order, and reordering to batch would change which widget is on top.
This is a real cost, knowingly paid.

**The hint is a dwell, not a hover.** A static with a hint shows it only after the cursor has
rested on it for a fixed interval, measured from the hover timestamp the window base stamps.

## The twins

| Twin | Role |
|---|---|
| [`UIStatic.cpp`](UIStatic.cpp.md) | The composite: texture, text, colour animation, transform animation, delayed hint, in a fixed draw order |
| [`UIStatic.h`](UIStatic.h.md) | Its declaration and public surface |
| [`UIStaticItem.cpp`](UIStaticItem.cpp.md) | The quad: screen conversion, normalised texture coordinates, mirroring, rotation, software clip, fan emission |
| [`UIStaticItem.h`](UIStaticItem.h.md) | Its declaration, and the mirroring vocabulary the layout format uses |
| [`UIAnimatedStatic.cpp`](UIAnimatedStatic.cpp.md) · [`UIAnimatedStatic.h`](UIAnimatedStatic.h.md) | The sprite-sheet player: a frame grid on one texture, stepped by elapsed time, cyclic or one-shot |
| [`UILanimController.cpp`](UILanimController.cpp.md) · [`UILanimController.h`](UILanimController.h.md) | Driving a widget from a named light-animation curve, with a channel-reinterpretation mode |
