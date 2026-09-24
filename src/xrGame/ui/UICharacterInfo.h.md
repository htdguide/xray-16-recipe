# src/xrGame/ui/UICharacterInfo.h

> Declares the portrait-and-particulars panel that every screen showing a person embeds.

**Needs** — [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md) · [`../../xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`../../xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md)
**Used by** — [`UIActorInfo.cpp`](UIActorInfo.cpp.md) · [`UIActorMenu.cpp`](UIActorMenu.cpp.md) · [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) · [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) · [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) · [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md) · [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UILogsWnd.cpp`](UILogsWnd.cpp.md) · [`UILogsWnd.h`](UILogsWnd.h.md) · [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md) · [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md) · [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md)
**Tier floor** — T2: polls a server-side record on a frame interval

## Purpose

Declares the surface implemented in [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md).

## `CUICharacterInfo`

The panel. It is the same widget in the inventory screen, the trade screen, the loot screen,
the talk screen and the personal terminal's statistics page, initialised from a different
layout document in each.

**Eighteen optional parts**, enumerated: portrait and its overlay, rank icon and overlay,
community icon and overlay, large community icon and overlay, and then nine text fields —
name, rank, community, reputation and relation, each with its own caption. Any part a document
omits is simply absent, and every operation guards.

- `InitCharacterInfo(position, size, document)` and the two convenience forms that load a
  document by name or read the geometry from a named element.
- `InitCharacter(entity id)` — bind to a character and fill everything.
- `InitCharacterMP(name, icon)` — the multiplayer form: a name and a portrait, nothing else.
- `ClearInfo()` — hide every part.
- `Update()` — refresh the relation line and the portrait tint on an interval.
- `get_actor_community(out our, out enemy)` — the player's faction pair in a faction war.
- `ignore_community(name)` — is this community excluded from icon display.

The panel identifies its subject by **entity identifier**, not by pointer, and re-resolves it
on every refresh — which is what lets it survive the character going offline.
