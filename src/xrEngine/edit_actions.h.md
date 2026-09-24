# src/xrEngine/edit_actions.h

> Declares the three kinds of link in a text-editor key's handler chain.

**Needs** — [`edit_actions.cpp`](edit_actions.cpp.md) · [`line_edit_control.h`](line_edit_control.h.md)
**Used by** — [`edit_actions.cpp`](edit_actions.cpp.md) · [`line_edit_control.cpp`](line_edit_control.cpp.md) · [`line_edit_control.h`](line_edit_control.h.md)
**Tier floor** — T3: three small declarations.

## Purpose

Declares the surface implemented in [`edit_actions.cpp`](edit_actions.cpp.md). Used only by
[`line_edit_control.cpp`](line_edit_control.cpp.md); the split is arbitrary and a rebuild
may fold it in.

## Exported units

- **`base`** — a chain link that owns and forwards to the rest of the chain.
- **`callback_base`** — a link guarded by a required modifier set; runs its callback and
  consumes the press when the requirement is met.
- **`key_state_base`** — a link that records one modifier as held and never consumes.
