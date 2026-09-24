# src/xrGame/stalker_sound_data.cpp

> The sidecar a stalker attaches to every sound it emits, so that whoever hears it learns
> who made it.

**Needs** — [`stalker_sound_data.h`](stalker_sound_data.h.md) · [`sound_user_data_visitor.h`](sound_user_data_visitor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_sound_data.h`](stalker_sound_data.h.md)
**Tier floor** — T2: a typed payload on a sound event.

## Purpose

Perception in this engine is event-driven: a sound *arrives* at a listener rather than being
polled for. That makes the sound event the only channel through which one creature learns
anything from another's noise, and it is why a sound carries an attachable payload at all.
This is the stalker's payload — it says "a stalker made this, and here it is".

The payload does not itself decide anything. It is handed to a visitor
([`stalker_sound_data_visitor.cpp`](stalker_sound_data_visitor.cpp.md)) that belongs to the
*listener*, and the visitor decides what the listener does with the knowledge. Splitting
emitter-identity from listener-reaction is what lets a creature react differently to the
same sound depending on who it is.

## State

```text
RECORD StalkerSoundData
  emitter : optional<reference to Stalker>   # cleared when the emitter dies or unloads
```

**Invariant** — the payload can outlive its emitter. A sound already in flight is not
retracted when the creature that made it is destroyed, so the reference must be clearable
and every use must tolerate its absence. This is the entire reason the record exists as a
separate object rather than as a stalker identifier copied into the sound.

## `accept`

**Contract** — offer this payload to a listener's visitor. Refuses silently when the emitter
is gone or is being destroyed; otherwise dispatches to the visitor.

```text
FUNCTION accept(visitor)
  IF emitter IS none OR emitter.being_destroyed
    RETURN                                  # the sound outlived its maker; nothing to say
  visitor.visit(self)
```

**Notes** — the *being destroyed* test is separate from the *gone* test and both are needed:
an entity part-way through teardown still has a live reference but must not be reasoned
about. A rebuild whose object lifetimes are managed differently still needs both answers —
"does the emitter exist" and "is it still a valid subject" — because the second is a game
state, not a memory-management artifact.

## `invalidate`

**Contract** — sever the link to the emitter. Called when the emitter is destroyed. After
this the payload is inert but still safe to deliver.

## `object`

**Contract** — the emitter. Callers must have established that it exists; the visitor does
so by only being reached through `accept`.
