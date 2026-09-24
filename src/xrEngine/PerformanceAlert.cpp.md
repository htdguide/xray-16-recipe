# src/xrEngine/PerformanceAlert.cpp

> Draws one performance complaint in big red text without disturbing the caller's own text cursor.

**Needs** — [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [`GameFont.h`](GameFont.h.md) · [`IPerformanceAlert.hpp`](IPerformanceAlert.hpp.md)
**Used by** — [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md)
**Tier floor** — T3: text output

## Purpose

One function, one decision: a complaint is emitted from the middle of a subsystem's own
statistics dump, using the same font object that dump is writing with, and the dump must
not notice. Everything here is about borrowing and returning.

## State

`Stateless` in itself; the cursor it advances lives in the record declared in the header.

## `Print`

**Contract** — saves the font's colour, cursor position and height; sets the alert colour
and **twice** the base height; writes one formatted line at the alert cursor; records where
the font ended up as the new alert cursor, so the next alert stacks below; then restores
all three saved properties. Allocates nothing, blocks never. Single-threaded: it is called
from the render phase only.

```text
FUNCTION print(self, font, format, args...)
  saved = (font.colour, font.position, font.height)
  font.colour = self.alert_colour
  font.position = self.alert_cursor
  font.height = self.font_base_size * 2
  font.write_line(format, args...)
  self.alert_cursor = font.position        # where the font advanced to
  font.colour, font.position, font.height = saved
```

**Invariants** — on return the font is byte-for-byte as it was on entry, because the caller
is part-way through laying out a column of its own numbers.

**Notes** — the doubled height is the whole design: alerts must be findable in a screen
already full of small statistics text, and the multiplier rather than an absolute size
keeps them proportional when the overlay font scales with resolution. The cursor advances
by *reading back* the font's position after writing rather than by adding a line height,
so a wrapped alert stacks correctly.
