# src/xrGame/alife_registry_container_composition.h

> The authoritative list of persistent per-character registries — which stores exist, what each one keys and holds, and the order they appear in a saved game.

**Needs** — [`alife_abstract_registry.h`](alife_abstract_registry.h.md) · [`alife_registry_container_space.h`](alife_registry_container_space.h.md) · [`InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md) · [`relation_registry_defs.h`](relation_registry_defs.h.md) · [`character_info_defs.h`](../xrServerEntities/character_info_defs.h.md) · [`encyclopedia_article_defs.h`](encyclopedia_article_defs.h.md) · [`GameTaskDefs.h`](GameTaskDefs.h.md) · [`game_news.h`](game_news.h.md) · [`map_location_defs.h`](map_location_defs.h.md) · [`actor_statistic_defs.h`](actor_statistic_defs.h.md) · [`PdaMsg.h`](PdaMsg.h.md)
**Used by** — [`alife_registry_container.h`](alife_registry_container.h.md)
**Tier floor** — T3: a data declaration

## Purpose

This is a data file written as code. It is the single definition of **what persistent
side-state the game keeps about characters**, and it is the field order of the largest
part of a saved game's non-entity data. Everything else in the registry-container family
is machinery for consuming this list.

The unifying shape: each registry is a map from a key (nearly always an entity identifier)
to a per-character collection. So "which information portions does character 42 know" is
one lookup in one registry, not a field on character 42's entity record. The separation is
deliberate — this state belongs to the *game*, not to the entity, and it must survive an
entity going offline, crossing a level boundary, or being destroyed and respawned.

## State

The nine registries, **in the order they are written to and read from a save**:

```text
RECORD RegistryList                       # key -> value, per registry
  known_information   : map<EntityId, list<InfoPortion>>
      # which authored information portions each character has learned; the
      # substrate the dialogue and task systems test their preconditions against

  character_relations : map<EntityId, RelationData>
      # how each character regards the others: personal goodwill on top of the
      # faction-level default

  known_contacts      : map<EntityId, list<TalkContact>>
      # for the player: everyone talked to, with the state of that conversation

  encyclopedia        : map<EntityId, list<ArticleId>>
      # for the player: which encyclopedia articles have been unlocked

  game_news           : map<EntityId, list<NewsItem>>
      # for the player: the news feed, mixing simulation-generated and scripted items

  specific_characters : map<text, int>
      # which authored character descriptions have already been handed out, so a
      # unique character is not instantiated twice. Keyed by section name, not by
      # entity identifier — the only registry that is

  map_locations       : map<EntityId, list<MapLocation>>
      # the markers on the player's map

  game_tasks          : map<EntityId, list<GameTask>>
      # the player's active and completed tasks, with per-objective state

  actor_statistics    : map<EntityId, list<StatSection>>
      # the player's tallies, for the statistics screen
```

Invariants:

- **The order above is the save format.** Reordering, inserting or removing an entry
  changes the byte layout of every save. There is no tag or length to resynchronize on.
- Eight of the nine are keyed by entity identifier, and in practice most of those only
  ever hold one key — the player's. They are keyed per character anyway because the
  design intends any character to be able to know information and hold relations; the
  player-only ones simply never grow a second key.
- `specific_characters` is keyed by the authored character section name instead, because
  its question is "has this *description* been used", which has no entity to hang on
  until it has been answered.

## Notes

The file is written as a sequence of macro invocations that both declare a registry's
type and push it onto the accumulating list, with a redefinition dance after each entry.
That is incidental; see
[`alife_registry_container_space.h`](alife_registry_container_space.h.md). A rebuild
writes this as nine lines of a table.

Two things a rebuilder should not copy:

- The convenience names for the last three registries collide in the original: the same
  name is redefined for the map-location registry, then for the game-task registry, then
  for the statistics registry, so only the last definition survives and the first two
  names are unreachable. Nothing uses them — every real caller selects a registry by its
  type through the container's selector — so this is dead text, not a defect with
  consequences. A rebuild gives each registry a distinct name.
- The commented-out fog-of-war registry in
  [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) is a registry that was
  removed from this list. Its absence is part of the current format.
