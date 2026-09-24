# src/xrGame/xrgame_dll_detach.cpp

> Fills and empties the game layer's shared, process-wide tables — the character, dialogue and reputation data every entity reads but none owns.

**Needs** — [`ai_space.h`](ai_space.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`ai/monsters/ai_monster_squad_manager.h`](ai/monsters/ai_monster_squad_manager.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`PhraseDialog.h`](PhraseDialog.h.md) · [`encyclopedia_article.h`](encyclopedia_article.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`specific_character.h`](../xrServerEntities/specific_character.h.md) · [`character_community.h`](character_community.h.md) · [`monster_community.h`](monster_community.h.md) · [`character_rank.h`](character_rank.h.md) · [`character_reputation.h`](character_reputation.h.md) · [`sound_collection_storage.h`](sound_collection_storage.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`script_properties_list_helper.h`](../xrServerEntities/script_properties_list_helper.h.md) · [`ui/UIInventoryUtilities.h`](ui/UIInventoryUtilities.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: constructs and releases process-wide tables in a fixed order

## Purpose

A dozen kinds of game data are *shared* rather than per-entity: what a community is, what a
rank means, the phrase graphs, the encyclopedia, the information flags the story turns on.
Each is loaded once from configuration and read by every entity that needs it. This file is
where that set is brought up and torn down as a whole, so that the order is stated in one
place rather than emerging from static-initialization order, which C++ does not define
across translation units.

That is the file's real subject: **the shared tables have a construction order and a
destruction order, and neither is the other reversed.**

## State

`Stateless.` — it constructs and releases state owned elsewhere.

## `init_game_globals`

**Contract** — load every shared table. Called once during game-layer startup. Reads
configuration and, for several tables, XML data files; blocks for the duration. Three of the
tables are **skipped entirely on a dedicated server**, because they exist only to be
displayed.

```text
FUNCTION init_game_globals()
  initialize the heads-up-display sound settings

  IF this process is not a dedicated server
    load the information-flag table       # variant selected by which game is being run
    load the encyclopedia articles        # variant selected by which game is being run
    load the phrase-dialogue graphs

  load the character-info table
  load the specific-character table
  load the character-community table
  load the character-rank table
  load the character-reputation table
  load the monster-community table
```

**Invariants** — the six always-loaded tables are loaded after the three conditional ones,
and the character-info table before the specific-character table. Communities, ranks and
reputations are independent of each other but all three must precede the first entity spawn,
because an entity's section names its community and rank by string and resolves them at
construction.

**Notes** — **Two of the tables load a different variant depending on which of the three
games is running.** The information flags and the encyclopedia articles are shaped
differently in the older titles than in the newest, and the selection is made from a global
that the startup sequence set from the mounted game data. A rebuild that supports all three
titles needs the same switch; one that supports only the newest can drop it.

The dedicated-server skip is not merely an optimization — those three tables pull in the
localization and the user-interface XML, which a headless build may not have. The rule to
carry across is: *presentation data is not loaded where there is no presentation*, and the
predicate is the server flag, not the absence of a graphics device.

## `clean_game_globals`

**Contract** — release everything the game layer owns process-wide. Called once, during
teardown. Releases in a hand-ordered sequence, not in reverse of construction.

```text
FUNCTION clean_game_globals()
  release the script property-list helper        # holds script-side references; first
  release the AI space                           # the pathfinder, the graphs, the planner pool
  release the object factory                     # the class-identifier registration table
  release the monster squad manager

  clear the story-identifier tables              # two: story ids and spawn story ids

  IF this process is not a dedicated server
    release the information-flag table: shared data, then the id-to-index map
    release the encyclopedia articles: shared data, then the id-to-index map
    release the phrase-dialogue graphs: shared data, then the id-to-index map

  release the character-info table:      shared data, then the id-to-index map
  release the specific-character table:  shared data, then the id-to-index map
  release the id-to-index maps of community, rank, reputation and monster community

  release the shared blood-decal appearance and the shared fire particles
  clear the cached character-info display strings
  release the sound-collection store
  clear the relation registry
```

**Invariants** — **each table is released in two steps, shared data before its
identifier-to-index map**, and the order within the pair is load-bearing: releasing a record
may consult its own index to find itself, and doing it the other way round leaves the
release walking a freed map.

The AI space is released before the object factory, because tearing down the AI space
destroys objects and destroying an object goes through the factory.

The script helper goes first because it holds references into the script virtual machine,
which is torn down on its own schedule; releasing it after the machine has gone is a
use-after-free.

**Notes** — The tables released unconditionally *include* four whose shared data is never
released, only their index maps — the communities, ranks and reputations keep their records
in static storage that outlives the process teardown. That asymmetry is a real inconsistency
in the original, not a decision: a rebuild should release both halves of every table
uniformly.

**Two entries here are not tables at all.** The shared blood-decal appearance and the shared
fire particles are renderer resources held statically by the living-entity class, so that a
thousand creatures share one. They are released here because there is no other point at
which the last creature's destruction is known to have happened. A rebuild with reference-
counted renderer resources does not need the manual release; the requirement that survives is
that renderer resources are released before the graphics device is.

The relation registry — who is hostile to whom — is cleared last because reputation and
community releases can still consult it.
