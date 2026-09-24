# src/xrUICore/InteractiveBackground — four looks, one drawn

> A container of four alternative backgrounds — enabled, disabled, highlighted, touched — over
> any widget type that can load a texture, of which exactly one is drawn.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Giving a control the visual-state model without giving it a fixed dressing. It is generic over
the *kind* of background: the same container serves a plain quad, a three-segment frame line or
a nine-slice frame, because all it requires is that the thing can load a texture.

The chapter uses it twice. A four-state quad background is what the plain-picture controls sit
on; a four-state frame-line background is what the track bar sits on.

## The load-bearing ideas

**State selects, it does not blend.** Exactly one of the four is drawn; there is no crossfade.
A rebuild that interpolates gets a different look from the shipped art, which was drawn as four
distinct images.

**The container forwards, and each backing kind adds what it alone can express.** Setting a
texture, a colour or a shader fans out to all four slots. Adjustments that only a plain quad
has — a texture offset, a stretch flag — are added by the quad specialisation, because the
generic container cannot name them.

**Four states, not three.** Enabled, disabled, highlighted and touched are distinct: a disabled
control is not a dimmed enabled one, and "touched" (held) is not "highlighted" (hovered). The
four-state button in [`Buttons/`](../Buttons/README.md) derives its condition the same way.

## The twins

| Twin | Role |
|---|---|
| [`UIInteractiveBackground.h`](UIInteractiveBackground.h.md) | The generic four-slot container over any texture-owning backing |
| [`UI_IB_Static.cpp`](UI_IB_Static.cpp.md) | Fanning the texture offset and the stretch flag out to all four slots |
| [`UI_IB_Static.h`](UI_IB_Static.h.md) | The quad-backed specialisation and the two adjustments the generic container cannot express |
