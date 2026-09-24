# src/xrGame/stalker_animation_script.h

> One entry in a stalker's script-animation queue: a motion the script layer asked for, plus how it should be played.

**Needs** — [`stalker_animation_script_inline.h`](stalker_animation_script_inline.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md)
**Used by** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_animation_script.cpp`](stalker_animation_script.cpp.md) · [`stalker_animation_script_inline.h`](stalker_animation_script_inline.h.md)
**Tier floor** — T3: a small immutable value record.

## Purpose

The script layer may tell a creature to play a named animation. That request is richer than
a motion identifier — it says whether the creature's hands are occupied during it, whether
the animation carries the creature through the world, and optionally exactly where it should
end up — so it is a record rather than a bare handle. Queueing those records is what lets a
script push a sequence and let it play out over many frames.

The record is written once at queue time and read until it is popped. It is a separate file
only because the animation manager is large; a rebuild may fold it in.

## State

```text
RECORD ScriptAnimation
  animation                : Motion
  hand_usage               : bool    # the animation uses the hands, so the weapon must be holstered
  use_movement_controller  : bool    # the animation drives the creature's position, not the movement manager
  local_animation          : bool    # a supplied transform is relative to the creature, not to the world
  transform                : optional<Transform>   # where the animation should land the creature
```

**Invariants**

- `transform` present means the caller supplied a destination; absent means "start from
  wherever the creature currently is", and the reader substitutes the creature's own
  transform. The two cases must stay distinguishable: a rebuild that defaults the field to
  identity silently teleports every script animation to the world origin.
- The record is immutable after construction. Nothing in the queue rewrites an entry; the
  manager pops entries and builds new ones.
- `local_animation` is only meaningful when `transform` is present.

## Exported units

- construction from a motion plus the three flags and an optional transform.
- copying — see [`stalker_animation_script_inline.h`](stalker_animation_script_inline.h.md);
  the copy is not the field-by-field default, and the reason is worth reading.
- `animation()`, `hand_usage()`, `use_movement_controller()`, `local_animation()` — plain
  readers.
- `has_transform()` — whether a destination was supplied.
- `transform(object)` — the destination, falling back to the object's own transform.
