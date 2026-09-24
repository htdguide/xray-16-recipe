# src/xrGame/script_monster_hit_info_script.cpp

> Exports the monster hit record as `MonsterHitInfo`, and a namespace object carrying the monster-facing constant tables.

**Needs** — [`script_monster_hit_info.h`](script_monster_hit_info.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`ai/monsters/monster_sound_defs.h`](ai/monsters/monster_sound_defs.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Two unrelated registrations share this file: the hit record, and an otherwise empty type
named `MonsterSpace` that exists purely as a namespace for constant tables the script
layer needs when talking to monsters.

## `script_register`

**Contract** — registers:

```text
class MonsterHitInfo
  who, direction, time        # all three readable and writable

class MonsterSpace            # no fields, no methods, no constructor
  MonsterSpace.sounds    = { sound_script }
  MonsterSpace.head_anim = { head_anim_normal, head_anim_angry,
                             head_anim_glad, head_anim_kind }
```

**Notes**

`MonsterSpace` is a *constant namespace wearing a class*: the binding layer used here can
attach constant tables to a class but has no first-class module object, so an empty type is
declared to hang them on. A rebuild whose script layer has plain namespaces should use one
and keep the dotted spellings, which shipped scripts write literally.

`sounds` exports exactly one value out of the monster sound vocabulary — the *scripted*
sound kind. Every other sound kind is chosen by the monster's own brain; only this one may
be requested from outside, which is why only this one is named.

`head_anim` selects which of four head poses a monster wears while a script drives it,
the creature-side equivalent of a facial expression during dialogue.
