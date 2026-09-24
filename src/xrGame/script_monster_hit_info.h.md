# src/xrGame/script_monster_hit_info.h

> The record a monster hands to script when asked who last hit it, from where, and when.

**Needs** — [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`script_game_object_use2.cpp`](script_game_object_use2.cpp.md) · [`script_monster_hit_info_script.cpp`](script_monster_hit_info_script.cpp.md)
**Tier floor** — T2: a plain record

## Purpose

A monster's brain records the last damage it took so that its script can react to it
without subscribing to a damage callback. This is the shape of that record as the script
sees it — a snapshot, not a live view: reading it twice in one frame gives the same
answer, and the monster overwrites it wholesale on the next hit.

## State

```text
RECORD MonsterHitInfo
  who       : optional<GameObject>  # the attacker's facade; none when nothing has hit yet
  direction : vector                # world-space direction the damage came from
  time      : int                   # engine clock reading at the moment of the hit;
                                    # zero means "never hit"
```

**Invariants**

- `time` of zero is the sentinel for "no hit recorded". A monster spawned and never struck
  reports zero, and scripts test that rather than testing `who`, because the attacker may
  have been destroyed since.
- `direction` defaults to the world's forward axis rather than zero, so a script that
  reads it before any hit still gets a usable unit vector instead of a degenerate one.

## Exported units

- construct — the empty record described above.
- `set(who, direction, time)` — overwrite all three fields together. They are always
  written as a set; a half-updated record would let a script attribute an old attacker to
  a new hit.
