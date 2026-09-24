# src/xrGame/ui/FractionState.h

> Declares the older, narrower sibling of the faction record — same idea, fewer fields, a
> different frozen script name.

**Needs** — [`FractionState.cpp`](FractionState.cpp.md) · [`FractionState_inline.h`](FractionState_inline.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`FractionState.cpp`](FractionState.cpp.md) · [`FractionState_inline.h`](FractionState_inline.h.md)
**Tier floor** — T3: a record with script-exported accessors

## Purpose

Declares the surface implemented in [`FractionState.cpp`](FractionState.cpp.md), with the
field accessors in [`FractionState_inline.h`](FractionState_inline.h.md).

This is **not** a duplicate of [`FactionState.h`](FactionState.h.md) that a rebuild may
delete: it is an earlier spelling — "fraction", in the original's non-native English — kept
alive because one game's shipped scripts bind to that name. See the implementation twin for
which fields differ.

## `FractionState`

- construct, or construct from a faction identifier.
- `update_info()` — refresh from the world and from script.
- Identity and presentation: identifier, display name, small icon, large icon, objective,
  objective description, location.
- Standing: the actor's goodwill.
- Numbers filled by script: member count, resource, power, `state_vs`, bonus.

No war-state slots: that is the difference that matters.
