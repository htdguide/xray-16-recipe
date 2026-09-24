# src/xrServerEntities/alife_space.h

> The alife simulation's shared vocabulary: the widths of every identifier that reaches disk or wire, the save-file chunk numbers, and the enumerations whose values are frozen by shipped configuration and scripts.

**Needs** — [Data: save games](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Data: level data — the spawn file](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`xr_object.h`](../xrEngine/xr_object.h.md) · [`CarSound.cpp`](../xrGame/CarSound.cpp.md) · [`CarWheels.cpp`](../xrGame/CarWheels.cpp.md) · [`CharacterPhysicsSupport.h`](../xrGame/CharacterPhysicsSupport.h.md) · [`DBG_Car.cpp`](../xrGame/DBG_Car.cpp.md) · [`GameObject.h`](../xrGame/GameObject.h.md) · [`Hit.cpp`](../xrGame/Hit.cpp.md) · [`Hit.h`](../xrGame/Hit.h.md) · [`Level.h`](../xrGame/Level.h.md) · [`Mincer.cpp`](../xrGame/Mincer.cpp.md) · [`PdaMsg.h`](../xrGame/PdaMsg.h.md) · [`ShootingObject.cpp`](../xrGame/ShootingObject.cpp.md) · [`ShootingObject.h`](../xrGame/ShootingObject.h.md) · [`Wound.cpp`](../xrGame/Wound.cpp.md) · _and 51 more_
**Tier floor** — T1: every type alias here is a declaration of an exact on-disk width.

## Purpose

Almost nothing in this file is code, and all of it is contract. It fixes the widths of the
identifiers the entity records are built out of, the container and chunk numbering of the
save file, and a dozen enumerations whose numeric values appear in shipped configuration
files and in shipped Lua. It is read by every twin in this chapter, so those twins can say
"an entity identifier" and mean something exact.

## State

### Identifier widths — the reason this file matters

```text
RECORD AlifeWidths
  class_identifier   : int (64-bit)   # the eight-character tag; see clsid_game.h
  entity_identifier  : int (16-bit)   # names an entity in saves, on the wire, in registries
  time               : int (64-bit)   # game time, in milliseconds
  event_identifier   : int (32-bit)
  task_identifier    : int (32-bit)
  spawn_identifier   : int (16-bit)   # index of the record in the level's spawn file
  terrain_identifier : int (16-bit)
  story_identifier   : int (32-bit)   # the script-visible name for a persistent entity
  spawn_story_id     : int (32-bit)   # the script-visible name for a spawn record
```

**Invariants**

- **The entity identifier is 16 bits and that is load-bearing.** It caps the world at
  65 535 entities, it is the width packed into every network message, and `0xffff` is the
  in-band "no entity" value — an entity with no parent stores `0xffff` in its parent field.
  A rebuild that widens it breaks the save format and the protocol together.
- **Both story identifiers use all-bits-set as "none"** (`-1` read as unsigned). Scripts
  compare against that value directly, so it cannot be re-spelled.
- **Time is 64-bit milliseconds** of game time, not real time, and is written to saves as
  such.

### Save file layout

```text
CONSTANT save_format_version = 7          # refused rather than guessed if mismatched
CONSTANT chunk_alife_data    = 0x0000     # the simulation's own state
CONSTANT chunk_spawn_data    = 0x0001     # the spawn records
CONSTANT chunk_object_data   = 0x0002     # the per-entity records
CONSTANT chunk_game_time     = 0x0005
CONSTANT chunk_registry      = 0x0009     # the script layer's serialized tables
CONSTANT spawn_file_name     = "game.spawn"
CONSTANT save_extension      = ".scop"    # ".sav" is also accepted, from the older games
CONSTANT section_prefix      = "location_"
```

**Invariants** — the chunk numbers are not contiguous; 3, 4, 6, 7 and 8 were used by
revisions that no shipped save contains. A rebuild must write the gaps as gaps, not
renumber. The version is compared for equality: the engine refuses a save it does not
recognize rather than attempting to read it, which is the right call for a format with no
per-field framing.

### Carrying capacity

```text
CONSTANT max_item_volume = 100      # the rucksack's capacity, in the data's own units
```

An arbitrary-looking number that is real: inventory items declare a volume in their
configuration section and the sum is checked against this. Changing it changes gameplay.

### Enumerations whose values are frozen

These reach shipped data, so the *numbers* matter and not just the names.

- **Hit type** — burn, shock, chemical burn, radiation, telepathic, wound, fire wound,
  strike, explosion, second wound (the knife's alternate attack), light burn, physical
  strike. Every weapon, anomaly and outfit section names one or more of these by *string*
  in the configuration, and the names are converted in
  [`alife_space.cpp`](alife_space.cpp.md). The numeric order is what an armour section's
  per-type resistance array is indexed by, so it is frozen by the data, not merely by the
  code.
- **Relation type** — friend, neutral, enemy, worst enemy. Used by the faction relation
  tables and exposed to scripts.
- **Meet action type** — attack, interact, ignore, go to smart terrain: what two offline
  entities do when the simulation decides they have met.
- **Influence type** — radiation, fire, acid, psi, electric: the five anomaly influences an
  outfit protects against, in the order the outfit's protection array is stored.
- **Condition restore type** — health, satiety, power, bleeding, radiation: the order of
  the actor's five regeneration rates.
- **Weapon priority type** — knife, secondary, primary, grenade: how a creature ranks the
  weapons it is carrying.
- **Combat result / combat action / combat type** — the outcomes and participants of an
  offline combat resolution.
- **Take type** — all, minimum, rest: how much of a stack an offline trade moves.
- **Weapon addon status** — disabled, permanently attached, attachable. Read directly out of
  a weapon's configuration section as a small integer, so the three values are data.

### Working collections

The file also names the aggregate shapes the simulation passes around — lists of entity
identifiers, of inventory item records, of weapon records, of schedulable records; maps from
entity identifier to record and from story identifier to record. These are conveniences, not
contracts; a rebuild names them however it likes.

## Notes

**Why the containers appear in a header of constants.** The alife simulation is spread
across many files that all need the same handful of collection shapes, and this file was the
common ancestor. The grouping is arbitrary; the widths and the chunk numbers above are not.
