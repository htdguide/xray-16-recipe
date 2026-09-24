# src/xrGame/relation_registry.h

> Declares the world's single opinion registry: who likes whom, how much, and what actions change it.

**Needs** — [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`relation_registry_defs.h`](relation_registry_defs.h.md) · [`relation_registry_inline.h`](relation_registry_inline.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md) · [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`ai_stalker_misc.cpp`](ai/stalker/ai_stalker_misc.cpp.md) · [`ai_trader.cpp`](ai/trader/ai_trader.cpp.md) · [`alife_human_abstract.cpp`](alife_human_abstract.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`map_manager.cpp`](map_manager.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · _and 9 more_
**Tier floor** — T2: a process-wide registry over the alife entity set

## Purpose

Declares the surface every part of the game asks "is he my enemy?" through: the AI's target
selection, the dialogue system, the map's marker colours, trading prices, and the scripts.
There is exactly one of these per running game, reached statically.

The implementation is split across three files by concern, which is worth preserving because
the three change for different reasons: storage and the goodwill arithmetic in
[`relation_registry.cpp`](relation_registry.cpp.md), the pending-fight bookkeeping in
[`relation_registry_fights.cpp`](relation_registry_fights.cpp.md), and the reaction rules for
player actions in [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md). The
derived queries, which are generic over anything that can report its identifier, faction,
rank and reputation, are in [`relation_registry_inline.h`](relation_registry_inline.h.md).

Two configuration section names are fixed here: the one carrying the relation thresholds and
the one carrying the per-action point awards. They are named in the shipped data and are
frozen.

Exported units:

- `relation between two characters` / `relation type from one to another` / `set relation
  type` — the three-valued friend/neutral/enemy view, derived. See the inline twin.
- `attitude` — the signed number the three-valued view is thresholded from. Derived.
- `goodwill` get, set, force-set and change — the *stored* personal opinion, the only part
  under direct control.
- `community goodwill` get, set and change — a faction's stored opinion of one character.
- `community relation` get and set — faction-to-faction standing, which lives on the faction
  table rather than here.
- `clear relations` — wipe one character's opinions, used when an identifier is recycled.
- `action` — apply the consequences of a kill, an attack, or help in a fight. The rules
  engine; see [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md).
- `fight register` and `update fight register` — record and expire in-progress fights so
  that "who helped whom" can be answered. See
  [`relation_registry_fights.cpp`](relation_registry_fights.cpp.md).
- `spot name` — the map-marker name for a relation type.
- `relation registry` / `clear relation registry` — the singleton's access and teardown.

## The action set

```text
ENUM RelationAction                # a bit per kind, because a stalker remembers the SET
  KILL               = 1           #   of things the player has done to it, not the last one
  ATTACK             = 2
  FIGHT_HELP_HUMAN   = 4
  FIGHT_HELP_MONSTER = 8
  SOS_HELP           = 16          # declared and never raised anywhere
```

**Notes** — the values are explicit powers of two so they can be accumulated into a per-
stalker flag set. `SOS_HELP` is declared, reserved a bit, and never used: no call site raises
it and the action handler has no case for it. A rebuild may drop it; it is recorded here
because its absence from the handler otherwise looks like an omission.

## `FightData`

```text
RECORD FightData                      # one in-progress fight
  attacker             : entity id
  defender             : entity id
  total_hit            : real         # accumulated damage; recorded, never read
  time                 : int          # last hit's timestamp — what expiry is measured from
  time_old             : int          # the previous hit's timestamp; recorded, never read
  attack_time          : int          # last time an ATTACK was scored for this attacker
  defender_to_attacker : relation type # the defender's opinion of the attacker when the
                                       #   fight began; recorded, and read only by code
                                       #   that is commented out
```

**Notes** — three of the seven fields are written and never read. The most interesting is
the last: it was meant to make a kill judged by how the victim felt *before* the fight
started, so that provoking a neutral into attacking you and then killing him counts as
killing a neutral. That intent is visible in the commented-out code and in the field's
existence, and the shipped behaviour instead judges the kill by the relation at the moment
of death. A rebuild should decide deliberately; the fork's behaviour is the simpler one.

## `RelationMapSpots`

**Contract** — maps a relation type to the name of the map marker used to draw a character of
that relation. Four types and a fallback; the worst-enemy type shares the enemy marker and
the fallback is the neutral marker, so out-of-range types degrade to neutral rather than
failing. Built lazily on first use.
