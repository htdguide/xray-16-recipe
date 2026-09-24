# src/xrServerEntities/smart_cast.h

> The registry of downcasts the engine performs often enough to be worth answering with a virtual call instead of a runtime type search.

**Needs** — [`smart_cast_impl0.h`](smart_cast_impl0.h.md) · [`smart_cast_impl1.h`](smart_cast_impl1.h.md) · [`smart_cast_impl2.h`](smart_cast_impl2.h.md) · [`smart_cast.cpp`](smart_cast.cpp.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`action_script_base_inline.h`](../xrGame/action_script_base_inline.h.md) · [`smart_cast.cpp`](smart_cast.cpp.md) · [`smart_cast_impl0.h`](smart_cast_impl0.h.md) · [`smart_cast_impl1.h`](smart_cast_impl1.h.md) · [`smart_cast_impl2.h`](smart_cast_impl2.h.md) · [`smart_cast_stats.cpp`](smart_cast_stats.cpp.md)
**Tier floor** — T1: it exists because the language's own runtime type query is slow; a tier whose type query is a table index does not need it.

## Purpose

"Is this record also an inventory item?" is asked tens of thousands of times a frame — by
the alife scheduler walking every record, by the inventory, by the hit system, by every
script call that receives a facade and wants a specific facet. The language's general answer
walks a type graph at run time and costs a hundredfold what the question deserves.

This file is the alternative: a **closed table of (target type, source type, method name)
triples**, declared once, from which a cast is compiled into a single virtual call. Every
type in the table publishes one small method per facet it can be asked about, each returning
itself or nothing. The cast becomes "call `is_an_inventory_item` on it and take the answer".

The table is the content. Everything else in the family is machinery for consuming it.

## How a cast resolves

```text
FUNCTION smart_cast<Target>(source) -> optional<Target>
  IF source is none
    RETURN none                       # a cast of nothing is nothing, not a failure
  IF Target is Source, or Target is a base of Source
    RETURN source                     # no check needed: the relationship is static
  IF a chain of registered conversions leads from Source to Target
    RETURN that chain applied          # one virtual call per link
  RETURN runtime_type_query(source)    # the slow general path
```

**Invariants** — **the maximum chain length is one.** Only a directly registered
(target, source) pair is answered by a virtual call; anything needing two hops falls back to
the general query. The search machinery in
[`smart_cast_impl1.h`](smart_cast_impl1.h.md) is written for longer chains and then clamped,
which is the file's largest piece of unexercised generality. The reason is not recorded;
compile time is the likely one, since the search is quadratic in the table's size and the
table has roughly seventy entries.

A cast must never change constness: casting away a read-only promise is rejected at build
time.

A debug build **replaces the whole scheme with the language's own query**, and additionally
asserts that the fast answer and the slow answer agree wherever both run. That
cross-check is the only thing keeping the table honest — a class that declares a facet
method and returns the wrong thing is otherwise undetectable.

## The table

Four families, and which of them exist depends on what is being built.

**Rendering and spatial facets** (game build only) — a visual asked for its skeleton, its
animated skeleton or its particle system; a skeleton asked for its animated form and back; a
spatially indexed thing asked for its renderable, light, sound-sensing or game-object facet.
These are the hottest entries: the renderer asks them per visible object per frame.

**Live game object facets** (game build only) — the facade downcast to its concrete type,
then to entity, living entity, inventory item, inventory owner, actor, weapon, magazine-fed
weapon, food, missile, zone, heads-up item, physics holder, input receiver, ammunition,
camera effectors, particle player, artefact, monster, stalker, script-driven entity,
restrictor, explosive, attachable item, attachment owner, vehicle holder, eatable item, and
base monster. Also the reverse direction — an inventory item, an inventory owner or an
attachment owner asked for the facade it belongs to — which is how a subsystem holding one
facet reaches another.

**Record facets** (every build, including the offline tools) — a record asked whether it is
an alife object, a dynamic object, ammunition, a weapon, a detector, a monster, a human, an
anomalous zone, a trader, a creature, a smart zone, an online/offline group, or a PDA. And
the mixin directions: a record asked for its inventory-item, trader, group or schedulable
mixin, and each mixin asked for the record it is part of.

**Structural facets** (every build) — a record asked for its visual, its motion, its shape
or its physics-skeleton part. These are the four interfaces of
[`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md) and the reason this file is
in this directory at all.

## Notes

**This file cannot have a normal include guard**, because it is included twice with
different meanings: once to *declare* the facet methods and build the table, once to
*define* the casts. The guard is defeated on purpose by the one file that does the second
pass ([`smart_cast.cpp`](smart_cast.cpp.md)). In a rebuild this is a generated file with two
outputs and no such trick.

**The sound-sensing facet is declared by hand** rather than through the table's usual entry,
because the declaring namespace has to be opened first. It is otherwise an ordinary entry.

**The facade's downcast to its concrete type is a plain static conversion**, not a virtual
call: the interface has exactly one implementation, so the check is known to succeed. That
is a strong assumption and it is written down only here.

**The cast's mechanism leaks into every class in the engine** — each one carries a method
per facet it participates in. That is the price, and it is the reason this table is closed:
adding an entry means editing the class as well. A rebuild whose type query is cheap should
delete the whole family and use it.
