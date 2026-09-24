# src/xrGame/ui/FactionState.h

> Declares the record the faction-war screen shows for one faction, and the script-visible
> names of its fields.

**Needs** — [`FactionState.cpp`](FactionState.cpp.md) · [`FactionState_inline.h`](FactionState_inline.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`FactionState.cpp`](FactionState.cpp.md) · [`FactionState_inline.h`](FactionState_inline.h.md) · [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) · [`UIRankFaction.cpp`](UIRankFaction.cpp.md) · [`UIRankFaction.h`](UIRankFaction.h.md)
**Tier floor** — T3: a record with script-exported accessors

## Purpose

Declares the surface implemented in [`FactionState.cpp`](FactionState.cpp.md), with the field
accessors in [`FactionState_inline.h`](FactionState_inline.h.md).

## `FactionState`

One faction's display state. The engine supplies the identity and the standing; everything
else is filled in by a script the engine calls.

- construct, or construct from a faction identifier.
- `update_info()` — refresh from the world and from script.
- `ResetStates()` — clear the five war-state slots.
- Identity and presentation: faction identifier, display name, small icon, large icon,
  objective, objective description, location.
- Standing: the actor's goodwill toward this faction.
- Numbers filled by script: member count, resource, power, bonus.
- Five war-state slots, each an icon name and a hint, addressed both by index and by five
  separately named accessor pairs.

## The two fixed counts

`war_state_count` is **five** and `bonuses_count` is **six**. Both are frozen by the shipped
screen layout, which has exactly that many slots, and by the script that fills them.
