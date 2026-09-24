# src/xrGame/game_type.h

> Declares the three free predicates that answer "which side of the client/server split am I on, and is this a single-player session".

**Needs** — _(none)_
**Used by** — [`BoneProtections.cpp`](BoneProtections.cpp.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`MainMenu.cpp`](MainMenu.cpp.md) · [`game_type.cpp`](game_type.cpp.md) · [`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry_fights.cpp`](relation_registry_fights.cpp.md) · [`UIMessagesWindow.cpp`](ui/UIMessagesWindow.cpp.md)
**Tier floor** — T3: three declarations

## Purpose

Declares the surface implemented in [`game_type.cpp`](game_type.cpp.md). It exists as its own
file because almost every entity in the game layer needs these three questions and nothing
else from the session machinery; a heavier dependency would drag the whole level and
persistent-game headers into every entity translation unit.

Exported units:

- `OnServer` — is the authoritative side live in this process.
- `OnClient` — is the local side live in this process.
- `IsGameTypeSingle` — is the running session the single-player game.
