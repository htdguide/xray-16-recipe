# src/xrServerEntities/smart_cast.cpp

> The single unit where the cast table is read a second time, to emit the cast bodies rather than the declarations.

**Needs** — [`smart_cast.h`](smart_cast.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast_impl2.h`](smart_cast_impl2.h.md)
**Tier floor** — T1.

## Purpose

The cast table in [`smart_cast.h`](smart_cast.h.md) is consumed twice. Everywhere in the
engine it is read in *declaring* mode: the table's entries become declarations, and the
table itself is assembled as a compile-time structure the cast machinery searches.
**Exactly one** unit reads it in *defining* mode, where each entry becomes the body of one
cast — a call to the facet method the entry names. This is that unit.

It is a file with no content of its own. Its whole job is: pull in the full definition of
every type the table mentions, then defeat the header's guard and read it again in the other
mode.

## Notes

**The include set is the table's dependency list made explicit** — visuals, the alife
enumerations, the hit record, the actor, monsters, stalkers, the UI window, zones, weapons,
camera effectors, and the record types. It is guarded so that the offline tools, which have
none of the live game types, pull in only the record half; the table itself is guarded the
same way, so the two agree.

**In a debug build this file compiles to nothing**, because the debug build uses the
language's own type query and needs no bodies at all.

A rebuild generates this from the table rather than writing it. There is no decision here to
preserve beyond "the bodies are emitted once, in one place, after every type is complete".
