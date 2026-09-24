# src/xrUICore/Windows — the tree node and the two stretchable frames

> The node every widget in the engine is, plus the two ways a small authored texture is
> stretched to dress a panel of arbitrary size.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Three types, one of which is the chapter.

The **window** is the node of the retained tree: parent and children, position and alignment,
visibility and enabled-ness, draw order and its reverse hit-test order, mouse and keyboard
capture, and the notification path a control uses to tell its owner something happened. Every
other widget in this chapter and in chapter 25 derives from it. It draws nothing itself — the
base window is invisible — so a subclass that wants to be seen overrides drawing and calls
back in to get its children painted.

The two **frames** are decoration, and they exist as separate types because they stretch
differently. A *frame line* slices a texture into three along one axis — first, centre,
second — and either stretches or tiles the centre; it is what dresses a scroll bar's trough,
an edit box, a list row's selection highlight. A *frame window* is the nine-slice: four
corners drawn at their authored size, four edges run along their axis, a centre fill. Both
can carry a lazily created title text.

## The load-bearing ideas

**Position is relative, clipping is not.** A window's absolute rectangle is the sum of its
ancestors' origins with its own size re-imposed — a child is positioned by its parent and
neither clipped nor resized by it. Containers that want clipping push a scissor explicitly.

**Alignment discards a coordinate on purpose.** The edge-pinned alignments read only the free
coordinate from the authored position and take the other from the *canvas* edge — not the
parent's edge, even for a nested window. Layout authors rely on that, so a rebuild must
preserve it rather than "correct" it to the parent.

**Capture is a chain, not a global.** A capturing widget's request walks up, each ancestor
capturing the link below it, so the root can reach the capturer in one hop per level. Capture
overrides position unconditionally, which is how a drag survives the pointer leaving the
widget. A displaced capturer is notified so it can abandon its gesture.

**Ownership travels with the detach.** A window marked auto-delete is destroyed by its
parent's detach and never independently; destroying one that is still parented and
auto-delete is a double claim.

**Show and enable are coupled in one direction.** Showing sets both, because a
hidden-but-enabled window would still answer input the player cannot see. Disabling alone is
how a visible control is greyed out.

## The twins

| Twin | Role |
|---|---|
| [`UIWindow.cpp`](UIWindow.cpp.md) | The tree node: geometry and alignment, attach/detach and ownership, draw and reverse hit-test order, capture, the four event channels, notifications, focus eligibility |
| [`UIWindow.h`](UIWindow.h.md) | Its declaration, and which operations a subclass may override |
| [`UIFrameWindow.cpp`](UIFrameWindow.cpp.md) · [`UIFrameWindow.h`](UIFrameWindow.h.md) | The nine-slice frame: four fixed corners, four run edges, a centre fill, plus an optional title |
| [`UIFrameLineWnd.cpp`](UIFrameLineWnd.cpp.md) · [`UIFrameLineWnd.h`](UIFrameLineWnd.h.md) | The three-segment stretched or tiled line, horizontal or vertical, plus an optional title |
