# src/xrUICore/InteractiveBackground/UI_IB_Static.cpp

> Fans the texture offset and the stretch flag out to all four background slots.

**Needs** — [`UI_IB_Static.h`](UI_IB_Static.h.md) · [`UIInteractiveBackground.h`](UIInteractiveBackground.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md)
**Used by** — [`UI_IB_Static.h`](UI_IB_Static.h.md)
**Tier floor** — T3.

## Purpose

Two fan-outs and nothing else. It is a separate file only because the generic container is a
template in the original; a rebuild folds both methods into that container as operations that
apply to whatever slots exist.

## `set_texture_offset`

**Contract** — sets the same texture offset on every loaded slot, including the current one.

## `set_stretch_texture`

**Contract** — sets the same stretch flag on every loaded slot.

**Invariants** — both iterate the slot array including the "current" alias, so the current
slot is written twice. Harmless, and worth noting only because a rebuild that makes "current"
a genuine reference rather than a slot must not write through it twice if the operation is
not idempotent. Both of these are.
