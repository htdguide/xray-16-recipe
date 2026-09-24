# src/xrGame/script_hit.h

> Declares the script-authored hit: a damage event a script builds by hand and applies to an entity.

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`script_hit.cpp`](script_hit.cpp.md) · [`script_hit_inline.h`](script_hit_inline.h.md) · [`script_hit_script.cpp`](script_hit_script.cpp.md)
**Tier floor** — T2: a plain record

## Purpose

A hit as the script layer builds one. The engine's internal damage event carries more —
a bone index, a surface, a weapon reference — but a script may only set these six fields,
and the receiving entity fills the rest in from context. This is the substance holder for
the type: the bodies are in [`script_hit_inline.h`](script_hit_inline.h.md) and are one
line each, and the script names are in
[`script_hit_script.cpp`](script_hit_script.cpp.md).

## State

```text
RECORD ScriptHit
  power      : real          # damage magnitude, in the same units as authored weapon damage
  direction  : vector        # world-space; drives the recipient's stagger and ragdoll impulse
  bone_name  : text          # resolved against the recipient's skeleton at apply time, not here
  draftsman  : optional<GameObject>   # who is blamed for the damage; none means the world
  impulse    : real          # physical push, independent of power: a shove can do no damage
  type       : int           # one of the damage kinds, see the script export
```

**Invariants**

- `power` and `impulse` are *independent*. The recipient's armour and its rigid body read
  different ones, so a hit that only staggers sets power to zero, and a hit that only
  wounds sets impulse to zero.
- `bone_name` is text, not a bone index, because the script does not know the recipient
  when it builds the hit. An empty name means "no particular bone", which the recipient
  resolves to its root.
- `direction` is not required to be normalized by the builder; the recipient normalizes.

## Exported units

- construct — a hit with the defaults listed in
  [`script_hit_inline.h`](script_hit_inline.h.md).
- construct from another hit — a copy, so a script can keep one template hit and vary a
  field per use.
- `set_bone_name(name)` — the only setter with a body; the rest are direct field access.

**Notes**

The type is copied by value into the engine's damage path, so a script may reuse and mutate
the same hit object after applying it. A rebuild that hands the recipient a reference
instead would change observable behaviour of shipped scripts that do exactly that.
