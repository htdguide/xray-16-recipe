# src/xrServerEntities/character_info_defs.h

> The five scalars that describe a person in this world — goodwill, class, reputation, rank, community — with their ranges and their "unset" values.

**Needs** — [`alife_space.h`](alife_space.h.md)
**Used by** — [`PDA.h`](../xrGame/PDA.h.md) · [`alife_registry_container_composition.h`](../xrGame/alife_registry_container_composition.h.md) · [`character_community.cpp`](../xrGame/character_community.cpp.md) · [`character_community.h`](../xrGame/character_community.h.md) · [`character_rank.cpp`](../xrGame/character_rank.cpp.md) · [`character_rank.h`](../xrGame/character_rank.h.md) · [`character_reputation.cpp`](../xrGame/character_reputation.cpp.md) · [`character_reputation.h`](../xrGame/character_reputation.h.md) · [`relation_registry.h`](../xrGame/relation_registry.h.md) · [`script_game_object.h`](../xrGame/script_game_object.h.md) · [`UIInventoryUtilities.h`](../xrGame/ui/UIInventoryUtilities.h.md) · [`character_info.h`](character_info.h.md) · [`specific_character.cpp`](specific_character.cpp.md) · [`specific_character.h`](specific_character.h.md) · _and 2 more_
**Tier floor** — T2.

## Purpose

Names the social attributes an entity record carries about a person, and — the part that is
actually contract — fixes what "no value" means for each, because several of them are
saved, several are read out of XML profiles, and the distinction between *neutral* and
*not set* drives the profile-inheritance rule in
[`character_info.cpp`](character_info.cpp.md).

## State

```text
RECORD CharacterAttributes
  goodwill    : int      # one person's feeling toward another
                         #   about -100 (hostile) .. +100 (friendly); neutral is 0
                         #   "unset" is the most negative representable value
  class       : text     # the character class name from the profile XML; "unset" is empty
  reputation  : int      # -100 (a bandit) .. +100 (honourable); neutral is 0
                         #   "unset" is the most negative representable value
  rank        : int      # 0 (a rookie) .. above 100 (a veteran)
                         #   "unset" is the most negative representable value
  community   : text     # the faction name; and separately
  community_index : int  # its position in the faction table; "unset" is -1
```

**Invariants**

- **"Unset" is not "neutral".** Goodwill, reputation and rank all have a real neutral value
  of zero and a separate unset value at the negative extreme. A profile that omits a rank
  inherits it from the specific character it names; a profile that sets it to zero does not.
  That rule is the only reason the two are distinguished, and losing it makes every
  unspecified character a rank-zero rookie.
- The extremes at ±100 are conventions the data honours, not bounds the code enforces.
  Scripts do push values outside them.
- A community is named by string in data and by index at run time; the index is into the
  faction table loaded from configuration, so it is only meaningful within one session.

## Notes

Goodwill is *directional and pairwise* — how A feels about B — while reputation is a single
value the whole world holds about one person. Both are stored as plain signed integers even
though the useful range is small; the width is not load-bearing here because these reach
saves through the record classes, which choose their own widths.
