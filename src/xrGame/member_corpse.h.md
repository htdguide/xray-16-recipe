# src/xrGame/member_corpse.h

> One squad member's corpse, and which surviving member has been assigned to react to it.

**Needs** — [`member_corpse_inline.h`](member_corpse_inline.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_corpse_manager.h`](agent_corpse_manager.h.md) · [`agent_corpse_manager_inline.h`](agent_corpse_manager_inline.h.md) · [`member_corpse_inline.h`](member_corpse_inline.h.md)
**Tier floor** — T3: a record

## Purpose

When a stalker in a squad dies, exactly one survivor should notice — walk over, speak, react
— rather than all of them at once or none of them. This record is the squad's bookkeeping for
that: the dead member, the survivor assigned to him, and when the death happened.

It is a free-standing record rather than a field on the corpse because the *assignment* is a
squad-level decision, made by the squad's coordinator, and the coordinator holds a list of
these.

## State

```text
RECORD MemberCorpse
  corpse   : Stalker      # the dead member
  reactor  : Stalker      # the survivor assigned to react; may be reassigned
  time     : int          # when the death was recorded, engine clock
```

**Invariant** — one reactor per corpse. The coordinator reassigns by writing a new one, which
is why that field alone is mutable; the corpse and the time are fixed at construction.

**Invariant** — the time is the moment the *record was made*, which is when the squad noticed,
not necessarily when the member died. It drives the reaction's timeout — a corpse nobody has
reacted to within a window stops being worth reacting to, because the fight has moved on.

## `CMemberCorpse` construction and accessors

**Contract** — construct from a corpse, a reactor and a time; read each back; replace the
reactor. Equality is against the *corpse* alone, so the coordinator finds a record by naming
the dead member — which is how it asks "have we already noticed this one".

**Notes** — comparing a record against a bare corpse is the whole reason equality is defined.
A rebuild with a map keyed by the corpse needs neither the comparison nor the search.
