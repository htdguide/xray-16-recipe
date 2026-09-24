# src/xrGame/ai/monsters/invisibility.h

> Declares the invisibility mixin: an energy budget that drains while hidden and recharges while shown, plus the flicker that covers the transition.

**Needs** — [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`invisibility.cpp`](invisibility.cpp.md)
**Tier floor** — T3: a scalar budget and a timer

## Purpose

Declares the surface implemented in [`invisibility.cpp`](invisibility.cpp.md). A creature that
can vanish inherits this alongside its own brain; the mixin owns the energy budget and the
visual flicker, and calls back into the creature at the three moments the creature cares
about.

## Exported units

- `activate` / `deactivate` — go hidden, go visible. Each starts a flicker.
- `energy` — the current budget, always in the unit interval.
- `active` — whether the creature is currently meant to be hidden.
- `full_energy` — whether the budget has recharged to its maximum.
- `set_manual_control`, `manual_activate`, `manual_deactivate`, `is_manual_control` — the
  script override: while manual, the budget stops moving on its own and only the explicit
  manual calls may flip the state.
- `reload` — reads the three tuned numbers from the creature's configuration section.
- `reinit` — resets to visible with an empty budget.
- `frame_update` — advances the flicker and the budget.
- `on_change_visibility`, `on_activate`, `on_deactivate` — the three hooks the implementing
  creature fills in. The first fires on every flicker edge, not once per transition.
