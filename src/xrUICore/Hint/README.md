# src/xrUICore/Hint — tooltips

> A box that sizes itself to its text, is nudged to stay on screen near the cursor, and appears
> only after the pointer has rested somewhere long enough.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Two halves. The **hint box** is the widget: it fits its own height to the wrapped text, and is
placed by the same fit-inside-a-rectangle routine the context menu uses, so it folds back at a
screen edge rather than being clipped. The **mixin** is what a widget inherits to own one and
the dwell rule that decides when to show it.

This is the per-screen tooltip. The process-wide pair that any widget can borrow for a frame
lives in [`Buttons/`](../Buttons/README.md) instead; the two coexist because a screen that
wants a styled tooltip of its own should not have to fight for the shared one.

## The load-bearing ideas

**A dwell, not a hover.** The hint appears only after the cursor has been inside the widget for
a fixed interval, measured from the timestamp the window base stamps when hover begins. Leaving
clears the stamp, so the timer restarts rather than resuming.

**The box sizes to the text.** Width is authored, height is derived from the wrapped line count
— which means the text engine has to have laid the text out before the box can be positioned.

**Text arrives raw or by key.** Both entry points exist because some hints are localization keys
and some are already-composed strings built from live game state.

**Drawn at the end of the frame**, with the cursor, so it is over every widget regardless of
where its owner sits in the tree.

## The twins

| Twin | Role |
|---|---|
| [`UIHint.cpp`](UIHint.cpp.md) | Fitting the box to its text, nudging it to stay on screen, and the dwell rule |
| [`UIHint.h`](UIHint.h.md) | The box and the mixin a widget inherits to own one |
