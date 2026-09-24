# src/xrEngine/ILoadingScreen.h

> The port through which the engine drives the loading screen it does not own.

**Needs** — [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`UILoadingScreen.cpp`](../xrGame/ui/UILoadingScreen.cpp.md) · [`UILoadingScreen.h`](../xrGame/ui/UILoadingScreen.h.md)
**Tier floor** — T3: a pure port, six calls, no layout and no data; nothing here needs manual memory or explicit layout

## Purpose

Level loading takes tens of seconds and must show progress, a level logo and rotating
hint text while it happens. The widget toolkit that draws all this lives in the game
module, not the engine, but the *schedule* — when it appears, when it is stepped, when it
is drawn — belongs to the level lifecycle. This file is the cut between the two: the
engine holds one of these and calls it; the game module supplies the implementation.

The split exists so the engine can bring a loading screen up before any game screens
exist, and so a dedicated server can supply a null implementation and pay nothing.

## State

`Stateless.` The interface owns no data; an implementor owns its own.

## `ILoadingScreen`

**Contract** — a loading screen is created once, initialized once after a graphics device
exists, then shown and hidden around each level load. While shown it is updated with a
completed/total stage pair and drawn every frame from inside the load loop, not from the
normal frame loop — the normal loop is not running during a load, which is the whole
reason this interface exists. All calls come from the thread that owns the graphics
device.

```text
INTERFACE LoadingScreen
  initialize()                       # after a graphics device exists
  is_shown() -> bool
  show(visible : bool)
  update(stages_completed : int, stages_total : int)   # progress fraction
  draw()                             # called from the load loop, not the frame loop
  set_level_logo(name : text)        # texture name for the level being entered
  set_stage_title(title : text)
  set_stage_tip(header : text, tip_number : text, tip : text)
```

**Notes** — the tip is passed as three separate strings rather than one composed line
because the implementor renders them in different fonts and positions; composing them
here would freeze a layout decision in the wrong module. `update` takes a pair rather
than a fraction so the implementor can also print "stage 4 of 11".
