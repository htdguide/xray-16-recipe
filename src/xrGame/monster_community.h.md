# src/xrGame/monster_community.h

> Declares the non-human faction: which species-group a creature belongs to, the team it fights on, and the square table of how any two groups feel about each other. Implemented in [`monster_community.cpp`](monster_community.cpp.md).

**Needs** — [`ini_id_loader.h`](ini_id_loader.h.md) · [`ini_table_loader.h`](ini_table_loader.h.md)
**Used by** — [`Entity.cpp`](Entity.cpp.md) · [`alife_creature_abstract.cpp`](alife_creature_abstract.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · [`entity_alive.h`](entity_alive.h.md) · [`monster_community.cpp`](monster_community.cpp.md) · [`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md)
**Tier floor** — T3: a declaration over two shared loaders

## Purpose

The non-human counterpart of [`character_community.h`](character_community.h.md). Wild
creatures are grouped by kind — dogs, flesh, boars, controllers and so on — and any two
groups have a fixed numeric attitude toward each other. That attitude is what decides
whether a pack of dogs attacks a boar, and it is authored, not emergent.

The shape is deliberately identical to the human faction type, and the reason to keep them
separate rather than merged is that they load from different configuration sections and
carry different extra fields: a human faction carries reputation and goodwill machinery, a
creature group carries only a team number.

Two ideas are fixed here.

**A community is a name outside and a dense index inside.** Configuration names groups by
text; everything at runtime uses a small integer, because the attitude lookup is a square
array indexed twice. The name-to-index mapping is built once at startup from the authored
list, and the index order is therefore the authored order — which makes the attitude table's
row and column order part of the frozen data.

**A community carries a team.** Several creature groups may share one team number, which is
what puts them on the same side of a multiplayer match and of the engine's coarse
friend-or-foe test without collapsing them into one group.

Exported units:

- `MONSTER_COMMUNITY_INDEX` / `MONSTER_COMMUNITY_ID` — the dense index and the authored
  name, with an out-of-band value marking "no community".
- `MONSTER_COMMUNITY_DATA` — one row of the community list: name, index, team.
- `MONSTER_COMMUNITY` — a creature's membership: settable by name or by index, readable as
  either, plus the team.
- `relation` — the attitude from one community to another, as a static two-argument form and
  as a one-argument form relative to this membership.
- `InitIdToIndex` / `DeleteIdToIndexData` — building and releasing the process-wide tables.
