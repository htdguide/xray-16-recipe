# src/xrGame/sound_memory_manager_inline.h

> Construction and the small accessors of the sound memory, including the threshold override that lets a creature be temporarily made deaf or sharp-eared.

**Needs** — [`sound_memory_manager.h`](sound_memory_manager.h.md)
**Used by** — [`sound_memory_manager.h`](sound_memory_manager.h.md)
**Tier floor** — T3: field assignment and one registration check

## Purpose

Separated from the declaration for C++ compilation reasons; fold into the type. Three of
these carry decisions.

## State

`Stateless.` — see [`sound_memory_manager.h`](sound_memory_manager.h.md).

## construction

**Contract** — binds the manager to its creature, its optional character brain and its
payload visitor, and sets the capacity to zero. Requires the creature and the visitor to
exist; the character brain may be absent. The record list is deliberately left unset here
— it is installed later by `set_squad_objects`, and every method asserts on it, so a
manager is unusable until that happens.

## `set_squad_objects`

**Contract** — points this creature's sound memory at a record list owned by somebody
else, normally its squad's. Takes no ownership and frees nothing on replacement.

**Notes** — this is the mechanism behind squad-wide hearing: members share one list, so a
sound one of them registers is immediately in every member's memory, tagged with the
contributing member in the record's squad mask. A creature with no squad is pointed at its
own private list. A rebuild must keep the aliasing, not copy: the records are refreshed in
place and copies would diverge within a frame.

## `priority` (register a sound type's rank)

**Contract** — associates a sound-type bitset with a priority rank. Asserts the type is
not already registered, so the table is write-once per type. Lower rank is more important.

## `set_threshold` / `restore_threshold`

**Contract** — override the current hearing threshold, and restore it to the configured
floor. Both validate the value is a finite number.

**Notes** — these exist so behaviour can make a creature situationally deaf (raise the
threshold while it is doing something noisy or absorbing) or situationally alert. They
bypass the decay in [`sound_memory_manager.cpp`](sound_memory_manager.cpp.md) only for the
instant of the call — the next sound event resumes decaying from whatever was set.

## `objects`, `sound`

**Contract** — expose the shared record list, and the record selected as most important by
the last update. The selected record exists only in checked builds; it is a debugging and
inspection aid, not a gameplay input, and a rebuild may omit it.
