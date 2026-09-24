# src/xrGame/saved_game_wrapper.h

> Declares the peek-at-a-save-file type: the four facts the load menu needs, without loading the game.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`saved_game_wrapper_inline.h`](saved_game_wrapper_inline.h.md)
**Used by** — [`Level_input.cpp`](Level_input.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md) · [`saved_game_wrapper_inline.h`](saved_game_wrapper_inline.h.md) · [`saved_game_wrapper_script.cpp`](saved_game_wrapper_script.cpp.md) · [`UIMMShniaga.cpp`](ui/UIMMShniaga.cpp.md)
**Tier floor** — T2: declares a small record read out of a compressed chunked file

## Purpose

The load screen lists saves with a date, a level name and a health bar. Producing those means
opening the save, which is a compressed snapshot of the entire world, and reading four things
out of it — without restoring anything. This type is that read.

The substance is in [`saved_game_wrapper.cpp`](saved_game_wrapper.cpp.md); the script export
is in [`saved_game_wrapper_script.cpp`](saved_game_wrapper_script.cpp.md).

## State

```text
RECORD SavedGameWrapper
  game_time    : int      # the in-world clock at the moment of saving
  level_id     : int      # which level the actor was on; all-ones when undetermined
  level_name   : text     # that level's display name; empty when undetermined, or the
                          #   localized error string when the id resolves to no level
  actor_health : real     # normalized; 1.0 when the save could not be read
```

**Invariants** — every field has a defined value after construction, including on every
failure path. The type never reports "I could not read this"; it reports plausible defaults
and an all-ones level identifier. Callers detect failure by that identifier, or by asking
the validity check first.

Exported units:

- construction from a save name — does the whole read.
- `saved_game_full_name` — name plus extension resolved against the saves root.
- `saved_game_exist` — does a save by this name exist, under either extension.
- `valid_saved_game` — two forms, over a stream and over a name: is this a save this build
  can load.
- `game_time` / `level_id` / `level_name` / `actor_health` — the four reads.
