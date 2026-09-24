# src/xrGame/ui/UIFactionWarWnd.h

> Declares the faction-war PDA page and its mirrored widget set.

**Needs** — [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`FactionState.h`](FactionState.h.md) · [`UIWarState.h`](UIWarState.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md). The
declaration is itself informative: **every member appears twice, once per side**, which is
the page's structure stated as a field list. A rebuild that represents a side as one record
and instantiates it twice reduces the class by half with no behavioural change.

Two counts are fixed here and are part of the contract with the shipped data: the number of
war stages comes from the faction-state vocabulary, and the number of bonus pips is six.

## Exported units

- **`CUIFactionWarWnd`**
  - constructed with the PDA's shared tooltip window, which every war-stage marker is given
    so that hovering a stage explains it.
  - `Init` — read the layout; **returns false when the document is absent**, which is how the
    tab is omitted in games without faction warfare.
  - `Show` — re-resolve the two factions and clear the stage markers.
  - `Update` — the throttled refresh.
  - `Reset` — clear cached factions, maxima and timing.
  - `InitFactions` — ask the character-info layer for the player's faction and its designated
    enemy.
  - `UpdateInfo` — one full refresh of both sides.
  - `UpdateWarStates` — fill and centre the stage row.
  - `ShowInfo` — show or hide the comparison half of the page as one unit.
  - `set_amount_our_bonus` / `set_amount_enemy_bonus` — light N of six pips.
  - `get_max_member_count` / `get_max_resource` / `get_max_power` — the three script calls.
