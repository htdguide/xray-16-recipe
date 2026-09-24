# src/xrGame/visual_memory_manager_inline.h

> The small accessors of visual memory, and the one of them that is not an accessor: aiming a creature's sighting list at its squad's.

**Needs** — [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Used by** — [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Tier floor** — T3: field reads and one conditional clear

## Purpose

Separated from the declaration for C++ compilation reasons; fold into the type. One of
these carries a decision.

## State

`Stateless.` — see [`visual_memory_manager.h`](visual_memory_manager.h.md).

## `set_squad_objects`

**Contract** — points this creature's remembered-sighting list at storage owned by
somebody else, normally its squad's. Takes no ownership and frees nothing on replacement.
Passing nothing detaches the manager, and **also clears the accumulator set**.

**Notes** — The coupled clear is the load-bearing line. Detaching happens when the owner
dies or leaves its squad, and a half-accumulated candidate is knowledge in flight: keeping
it would let a creature re-attached to a new list instantly notice something it had been
slowly working out before it died. The two sets are therefore attached and detached
together, and a rebuild that treats the accumulators as independent scratch will leak
perception across a squad change.

## `objects`, `raw_objects`, `not_yet_visible_objects`

**Contract** — read the three sets: the remembered sightings, this pass's geometric
candidates, and the accumulators. All three are borrowed views for inspection by the brain,
the squad layer and the debug overlay; none transfers ownership.

## `visibility_threshold`, `transparency_threshold`

**Contract** — the two thresholds of the *currently selected* tuned profile, not of a fixed
one. Reading them therefore changes answer when the owner switches between its free and its
danger profile, which is the point: a creature in danger both notices faster and sees
through less.

## `enabled`, `enable(flag)`

**Contract** — read and write the sensing switch. Turning it off stops the sensing pass but
preserves the remembered set, so sensing resumes from what was known rather than from
nothing.
