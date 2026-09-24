# src/xrGame/script_sound_action_inline.h

> The sound channel's seven constructors and its setters — where the mode and the goal kind are decided by statement order.

**Needs** — [`script_sound_action.h`](script_sound_action.h.md) · [`script_sound.h`](script_sound.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for [`script_sound_action.h`](script_sound_action.h.md). Three decisions live here:
how each constructor lands on its goal kind, which setters clear the completion flag and
which only clear the started flag, and what the monster and trader forms leave unset.

## The four general constructors

**Contract** — each writes the loop flag directly, then runs setters. The bone forms call
the bone setter *before* the sound setter, so the sound setter's tag wins and the goal is
*attached*; the position forms call the position setter *after* the sound setter, so the
position setter's tag wins and the goal is *positioned*.

```text
FUNCTION construct(sound, bone_name, offset, angles, looped, kind)
  looped = looped
  set_bone(bone_name)
  set_position(offset)      # tags positioned
  set_angles(angles)
  set_sound(sound)          # tags attached   <- wins
  set_sound_type(kind)

FUNCTION construct(sound, position, angles, looped, kind)
  looped = looped
  set_sound(sound)          # tags attached
  set_position(position)    # tags positioned <- wins
  set_angles(angles)
  set_sound_type(kind)
```

**Invariants** — `looped` is written as a field, not through a setter, so it does not clear
the started flag. Harmless at construction, and a trap for anyone who later adds a
`set_looped`: it would need to clear it.

Both shapes exist twice, once taking a sound name and once taking a script-owned sound
handle to copy the name from.

## Monster constructors

**Contract** — record the monster sound kind, record the delay (the one-argument form uses
−1, meaning the creature's own default), and clear the completion flag. Nothing else is
touched: no sound name, no goal kind, no position.

**Invariants** — the goal kind is left as *none*. That is correct and load-bearing: a
monster vocalization is placed by the creature, from its own authored emitter bone, and a
goal kind here would override that.

## Trader constructor

**Contract** — set the bone, set the sound, clear the completion flag, record the head
animation. No position, no angles, no loop flag, no sound kind.

**Notes**

The completion flag is cleared explicitly even though `set_sound` already does it, because
`set_sound` may *set* it — a missing sound file marks the channel finished on the spot,
see [`script_sound_action.cpp`](script_sound_action.cpp.md). Clearing it afterwards here
means **the trader form plays a missing sound forever instead of skipping it**. That is
almost certainly a bug rather than a decision; a rebuild should not copy it.

## Setters

**Contract** —

```text
set_sound(script_sound)  : copy its name; tag attached;   clear started; clear completed
set_position(p)          : write; tag positioned;         clear started
set_bone(name)           : write;                         clear started
set_angles(a)            : write;                         clear started
set_sound_type(kind)     : write;                         clear started
initialize               :                                clear started
```

**Invariants** — only the sound setters touch the completion flag. Changing where a sound
comes from or what it sounds like to a creature re-arms the channel for replay but does not
un-finish it; changing *which sound* does. A rebuild that clears both everywhere changes
when a queued action reports done.
