# src/xrGame/sound_memory_manager.h

> Declares a creature's memory of sounds it has heard: what was recorded, how loud a noise has to be to register, and how that threshold decays.

**Needs** — [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) · [`memory_space.h`](memory_space.h.md) · [`sound_memory_manager_inline.h`](sound_memory_manager_inline.h.md) · [`sound_user_data_visitor.h`](sound_user_data_visitor.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) · [`memory_manager.cpp`](memory_manager.cpp.md) · [`memory_manager.h`](memory_manager.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) · [`sound_memory_manager_inline.h`](sound_memory_manager_inline.h.md)
**Tier floor** — T2: per-event work on the simulation thread, serialized into the save format

## Purpose

Declares the surface implemented in
[`sound_memory_manager.cpp`](sound_memory_manager.cpp.md). This is one of the three
*memory managers* a creature owns — sound, vision and hit — and the one driven purely by
events rather than by a per-frame scan.

Exported units:

- `feel_sound_new` — the hearing event: a sound reached this creature.
- `update` — per-frame maintenance and, in a checked build, selection of the most
  important remembered sound.
- `reload(section)` / `reinit` / `Load` — configuration and lifecycle.
- `save` / `load` — persistence, with a deferred-resolution path for entities that are not
  spawned yet.
- `remove` / `remove_links` — forget one record, or every record naming an entity.
- `enable(object, flag)` — mute or unmute an entity's records without deleting them.
- `set_squad_objects` — point this creature's memory at a *shared* record list.
- `set_threshold` / `restore_threshold` — override and restore the hearing threshold.
- `objects` / `sound` — read the records, and the currently selected one.
- `on_requested_spawn` — the callback fired when a deferred entity finally spawns.

## State

```text
RECORD sound_object                  # one remembered sound (defined in memory_space)
  object        : optional<game object>   # none means "the world made this noise"
  object_params : (level vertex, position)  # where the noise came from
  self_params   : (level vertex, position)  # where the hearer was when it heard
  sound_type    : enum ai_sound (bitset)
  power         : real
  squad_mask    : int (bitset)        # which squad members contributed this record
  level_time    : int                 # when it was last refreshed
  enabled       : bool

RECORD sound_memory_manager
  object            : creature          # the hearer
  stalker           : optional<stalker> # set only for the character brain; see Notes
  visitor           : payload visitor   # reads the AI payload of every sound heard
  sounds            : list<sound_object>  # NOT owned — see set_squad_objects
  priorities        : map<sound type, int>  # lower number = more important
  max_sound_count   : int               # capacity; the list evicts, never grows past it
  delayed_objects   : list<(entity id, sound_object)>   # awaiting a spawn

  last_sound_time      : int (world clock)
  sound_threshold      : real           # current bar a sound must clear
  min_sound_threshold  : real           # the floor it decays back to
  self_sound_factor    : real
  sound_decrease_quant : int (ms)       # the decay's time unit
  decrease_factor      : real           # the decay's per-quantum multiplier

  weapon_factor, item_factor, npc_factor,
  anomaly_factor, world_factor : real   # per-category loudness weighting
```

**Invariants** — `sounds` is a borrowed pointer, not owned storage: a creature in a squad
has it aimed at the squad's shared list so that one member hearing something enters it
into everyone's memory. Nothing here frees it, and every method asserts it is set.
`priorities` maps a sound-type bitset to a rank in which **a lower number is more
important** — the same inverted convention as the sound player's priorities.

**Notes** — the manager holds both a generic creature reference and an optional character
reference to the same entity. The second is how it asks squad and enemy questions that
only a character brain can answer; when it is absent the creature is a monster, which has
no squad and no enemy selection, and the code takes simpler paths. A rebuild should model
this as an optional capability rather than a second pointer to the same thing.
