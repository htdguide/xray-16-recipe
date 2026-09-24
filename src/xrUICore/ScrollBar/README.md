# src/xrUICore/ScrollBar — position within a range

> Two buttons, a trough and a thumb. The thumb's size means "how much of the content fits",
> its position means "where in the content we are", and both are integers.

Part of [chapter 15](../README.md).

## What this directory is responsible for

A scroll bar is assembled from four widgets it owns — a decrement button, an increment button,
a draggable box, and a framed background — and it publishes one thing: a scroll position, sent
to its owner as a notification whenever it changes.

Two variants exist. The ordinary one **resizes its thumb** to reflect how much of the content
is visible, and is used wherever the content's extent is known. The fixed variant keeps the
thumb at a constant size and uses a four-state button for it, which is how the personal-digital-
assistant screens in the shipped games look; it overrides the three geometry routines that
would otherwise resize the thumb, and its width and height setters are deliberately inert.

The **scroll box** is the draggable thumb itself: a three-segment frame that does nothing but
report the pointer actions it receives to the scroll bar.

## The load-bearing ideas

**The model is four integers, and the thumb is derived from all four.** A minimum, a maximum, a
page size and a position. The scrollable extent is `max − min − page + 1`, floored at one — so
content that fits leaves exactly one reachable position rather than a degenerate range. A
rebuild that stores a fraction instead will not reproduce the shipped stepping, which is
integral.

**There are three ways to move it and they are not the same.** The end buttons step by a step
size; the trough pages by the page size; the thumb drags continuously and maps its pixel
position back to the integer range. Only the drag is lossy, and it clamps rather than wrapping.

**A held end button repeats after a delay.** One delay, not the spinner's two.

**Relevance is decided, not assumed.** The bar knows whether it is meaningful for the current
content, and the owner may show or hide it on that basis — the fixed variant declares itself
always relevant, which is why it is always visible in the screens that use it.

**Orientation is a construction parameter.** The same type is horizontal or vertical, chosen at
initialization along with a profile name that selects the textures. The profile is what lets
three games' art dress the same control.

## The twins

| Twin | Role |
|---|---|
| [`UIScrollBar.cpp`](UIScrollBar.cpp.md) | The four-integer model, the three ways to move, the thumb geometry, the repeat on held buttons, and the position notification |
| [`UIScrollBar.h`](UIScrollBar.h.md) | Its declaration and the profile-driven construction |
| [`UIFixedScrollBar.cpp`](UIFixedScrollBar.cpp.md) | The constant-size thumb: the three geometry routines overridden and the resize setters made inert |
| [`UIFixedScrollBar.h`](UIFixedScrollBar.h.md) | Its declaration |
| [`UIScrollBox.cpp`](UIScrollBox.cpp.md) · [`UIScrollBox.h`](UIScrollBox.h.md) | The draggable thumb: a three-segment frame that forwards its pointer actions |
