# src/xrUICore/Cursor — the one pointer position

> A single position in canvas units, fed from two different input sources depending on whether
> the pointer is captured, and drawn after everything else in the frame.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Owning the interface's pointer position, its visibility, and its drawing. Every hover test,
every pointer event and every tooltip placement in the chapter reads from here, so there is
exactly one of these and it lives on the module's runtime object.

## The load-bearing ideas

**The position is in canvas units, not pixels.** Like everything else in the chapter, the
cursor works in the fixed virtual canvas, so a hit test against a widget rectangle needs no
conversion.

**Two feeds, chosen by capture mode.** With the pointer captured — the in-game case, where the
OS pointer is hidden and the mouse aims the camera — position is accumulated from relative
deltas and clamped to the canvas. With the pointer free — the menu case — it is taken from the
absolute OS position and converted. A rebuild that keeps only one of these loses either camera
aim or menu pointer accuracy, depending which it drops.

**The cursor is drawn last, and so are the deferred hints.** Tooltips and hint boxes are not
drawn where they are triggered; they are deferred to the very end of the frame so that they
appear above every widget regardless of tree position. That deferral is the only exception to
the "z-order is list position" rule in the chapter, and it exists because a tooltip's owner is
usually deep in a tree.

**The delta is readable.** Widgets that drag — the scroll view's pad, the track bar's slider —
read the per-frame movement rather than differencing positions themselves.

## The twins

| Twin | Role |
|---|---|
| [`UICursor.cpp`](UICursor.cpp.md) | The position in canvas units, the two feeds, and the end-of-frame draw of cursor and deferred hints |
| [`UICursor.h`](UICursor.h.md) | Its declaration and visibility control |
