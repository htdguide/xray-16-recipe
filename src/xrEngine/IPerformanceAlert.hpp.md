# src/xrEngine/IPerformanceAlert.hpp

> The port a statistics dump writes through when a number it just printed is bad enough to shout about.

**Needs** — [`GameFont.h`](GameFont.h.md)
**Used by** — [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [`Render.h`](Render.h.md)
**Tier floor** — T3: two calls, no state

## Purpose

Every subsystem that dumps per-frame statistics is also the only code that knows which of
its numbers constitute a problem — the renderer knows what a bad draw-call count is, the
scheduler knows what a blown time budget is. Rather than teach a central overlay every
subsystem's thresholds, each dump routine receives an alert sink and writes its own
complaints into it. The sink renders them apart from the normal statistics, large and
red, so they are visible without reading the wall of numbers.

The port is separate from its one implementation so that passing *nothing* is a legal way
to say "statistics on, complaints off", which is what the shipping configuration does.

## State

`Stateless.`

## `IPerformanceAlert`

**Contract** — `reset` is called once per frame before any dump runs and returns the
sink's cursor to its starting position. `print` appends one formatted complaint line;
formatting is the caller's, in the same variadic style as the rest of the engine's text
output. Printing borrows the caller's font, changes colour and size for the duration, and
restores every font property it touched — the caller must see the font exactly as it left
it, because the caller is mid-way through its own column of output.

```text
INTERFACE PerformanceAlert
  reset()
  print(font : Font, format : text, args...)
```

**Invariants** — after `print` returns, the font's colour, position and height are what
they were on entry.
