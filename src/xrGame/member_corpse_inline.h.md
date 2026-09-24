# src/xrGame/member_corpse_inline.h

> Construction and accessors for the squad's corpse-reaction record.

**Needs** — [`member_corpse.h`](member_corpse.h.md)
**Used by** — [`member_corpse.h`](member_corpse.h.md)
**Tier floor** — T3: field reads

## Purpose

Carries the corpse record's bodies out of the declaration. A rebuild folds them in; the
substance is in [`member_corpse.h`](member_corpse.h.md).

## State

`Stateless.`

## Construction · `corpse` · `reactor` · `time` · equality

**Contract** — as declared: all three fields are set at construction, the reactor may be
replaced afterwards, and equality compares only the corpse so that a record can be found by
naming the dead member.
