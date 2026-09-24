# src/xrGame/BottleItem.h

> Declares the breakable drinkable implemented in [`BottleItem.cpp`](BottleItem.cpp.md).

**Needs** — [`FoodItem.h`](FoodItem.h.md)
**Used by** — [`BottleItem.cpp`](BottleItem.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CBottleItem`, a consumable that shatters when hit hard. Substance is in
[`BottleItem.cpp`](BottleItem.cpp.md).

Exported units:

- `CBottleItem` — a food item holding a shatter particle name and a shatter sound.
- `Load` — read both, each optional.
- `Hit` — above a fixed damage threshold, send a breakage event at itself.
- `OnEvent` — react to that event.
- `BreakToPieces` — the sound, the particles, and — only on the authoritative side —
  self-destruction.
