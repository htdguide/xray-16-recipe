# src/xrGame/script_sound_action_script.cpp

> Exports the sound channel as `sound`, and the engine's whole AI-perception sound vocabulary as `snd_type`.

**Needs** — [`script_sound_action.h`](script_sound_action.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`ai/monsters/monster_sound_defs.h`](ai/monsters/monster_sound_defs.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Two registrations: the sound channel, and an empty type used as a namespace for the
**AI sound kind** table — the perception attributes every sound in the game is tagged with.
The second is by far the more important of the two, because that table is the vocabulary
the senses system reasons in.

## `sound`

**Contract** — registers a class named `sound` with a constant table of monster
vocalizations, seventeen constructors and seven methods.

```text
sound.type = { idle, eat, attack, attack_hit, take_damage,
               die, threaten, steal, panic }

constructors: the general form (sound name or sound handle) x (bone or position)
              x optional placement, angles and loop flag — fourteen arities;
              the monster form (type) and (type, delay);
              the trader form (sound name, bone name, head animation)

methods: set_sound(name), set_sound(sound_handle), set_sound_type(kind),
         set_bone(name), set_position(vector), set_angles(vector), completed()
```

**Notes**

The monster vocalization names are *roles*, not files: a creature's configuration binds
each role to its own authored sounds, so `attack` means something different for a dog and
for a burer. That indirection is why this table is short while `snd_type` is long.

## `snd_type`

**Contract** — registers an empty type named `snd_type` carrying one constant table,
`sound_types`, with forty-six entries. The values are **composable**: the vocabulary is a
two-level classification where a broad category and a specific action combine.

```text
categories : no_sound, weapon, item, monster, anomaly, world
actions    : pick_up, drop, hide, take, use, shoot, empty, bullet_hit,
             reload, die, injure, step, talk, attack, eat, idle,
             object_break, object_collide, object_explode, ambient
combined   : item_pick_up, item_drop, item_hide, item_take, item_use,
             weapon_shoot, weapon_empty, weapon_bullet_hit, weapon_reload,
             monster_die, monster_injure, monster_step, monster_talk,
             monster_attack, monster_eat, anomaly_idle,
             world_object_break, world_object_collide, world_object_explode,
             world_ambient
```

**Invariants**

- A combined name is the *combination* of its two parts, not a separate value. The listener
  side matches on either half: a creature can be interested in every weapon sound, or in
  every shot from any source, or in exactly a weapon shot. A rebuild must keep the
  composition — implementing these as forty-six opaque constants breaks every partial
  match in the senses system.
- The pre-combined names are exported because the binding layer cannot combine constants at
  script load time cheaply and shipped scripts use the combined spellings directly. A
  rebuild may export the two halves and let scripts combine them only if it also keeps the
  combined names as aliases.
- `no_sound` is the zero: a sound tagged with it is audible but produces no perception
  event, and it is the default everywhere.

**Notes**

The empty type exists only to hang the table on, the same trick as
[`script_monster_hit_info_script.cpp`](script_monster_hit_info_script.cpp.md). A rebuild
with real namespaces should use one and keep the `snd_type.` prefix in the spelling.
