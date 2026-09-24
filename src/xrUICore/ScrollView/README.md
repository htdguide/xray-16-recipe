# src/xrUICore/ScrollView — the scrolling container

> Children of any height, stacked vertically inside a clipped viewport, moved by sliding an
> inner pad rather than by moving each child.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The general-purpose vertical scroller. Unlike the row-paged list in
[`ListWnd/`](../ListWnd/README.md), it makes no assumption that its children are a uniform
height: it stacks them by their own sizes, with authored indents on all four sides and a
configurable interval between them, and it can scroll to an arbitrary offset rather than to a
whole row.

It is the container under the inventory, the trade screen, the dialogue list and most of the
long text in the game.

## The load-bearing ideas

**Scrolling moves one window, not many.** Children are attached to an inner *pad* window whose
height is the content's total height; scrolling sets the pad's vertical position to the negative
of the scroll position. Every child therefore keeps its own coordinates and never learns it has
moved. This is the whole mechanism, and it is what makes scrolling constant-cost regardless of
content.

**Layout is lazy and flag-driven.** Adding, removing or resizing a child raises a recalculate
flag; the relayout happens on the next draw, update or size query. A child that changes size
notifies its parent, and the view recognises the notification only for its own pad's children.

**Drawing clips and skips.** The viewport pushes a scissor rectangle, and the view remembers
the index range of children that intersected it last time. While that range stays valid only
those children are drawn; any scroll or relayout invalidates it and the next draw rescans
until it finds the first intersecting child and stops at the first that does not. The scan
relies on children being stacked in order, which the layout guarantees.

**Focus drags the view.** When navigation focus lands on a widget somewhere inside, the view
maps that widget back to the direct child of the pad containing it and scrolls that child into
sight — and if the scroll actually moved, it warps the cursor onto the widget so pointer and
focus do not disagree. That mapping is the tree query "which of my rows am I in".

**The pad can be dragged directly.** Holding the primary button and moving over the content
slides the pad by the cursor delta, clamped to the content extent — a touch-style drag that
coexists with the scroll bar.

**Two orderings and two anchors are authored.** The content can be laid out in reverse child
order, and it can be anchored to the bottom rather than the top, because chat-like and
list-like screens want opposite behaviour. An optional comparison function sorts children
before layout.

**The scroll bar may be either kind, with a fallback.** The view asks for the fixed-thumb bar
first; if its profile cannot be initialized it falls back to the ordinary bar, logging the
substitution. A rebuild shipping one kind of art only can drop the fallback.

## The twins

| Twin | Role |
|---|---|
| [`UIScrollView.cpp`](UIScrollView.cpp.md) | The pad, the lazy relayout, the scissor-clipped visible-range draw, the drag, the focus-follows-scroll rule, and the scroll-bar fallback |
| [`UIScrollView.h`](UIScrollView.h.md) | Its declaration, the indents and flags, and the item accessors scripts use |
