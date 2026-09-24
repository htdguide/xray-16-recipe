# src/xrEngine/ObjectDump.cpp

> Renders a game object as text, so a crash message can say what broke rather than where.

**Needs** — [`ObjectDump.h`](ObjectDump.h.md) · [`xr_object.h`](xr_object.h.md) · [`device.h`](device.h.md)
**Used by** — [`ObjectDump.h`](ObjectDump.h.md)
**Tier floor** — T3: pure formatting

## Purpose

When an assertion fires deep in the object system, the pointer in the message is useless
and the object is about to be freed. These routines capture the object's identity, state
and pose as text at the moment of failure so the message carries something a human can act
on. They exist as a separate file so the same rendering is used by every assertion site
rather than each inventing its own.

Development builds only.

## State

`Stateless.`

## The dumps

**Contract** — each takes an object that may be absent and returns text; an absent object
yields either a marker line or empty text rather than failing, because these run on a path
that is already failing. None allocates persistently, none blocks, none mutates the object.

- **identity** — the instance's own name, the configuration section it was spawned from,
  and the visual asset's name when it has one. This is the line that actually identifies
  the object to a human, because the section name is what a modder searches for.
- **properties** — the object's packed state bits rendered as named booleans (enabled,
  visible, destroy-pending, network-local, network-ready, needs server update, "crow",
  pre-destroy), its network identifier and activity counter, and three frame stamps: the
  frame it was last updated in, the frame it was last updated as a crow, and the debug
  stamp. Those three are printed alongside the device's current frame and global time
  *specifically* so a reader can see how stale the object is — an object whose update
  stamp is many frames behind the device frame is one the scheduler has been starving, and
  that is the most common cause of the bugs this dump is read for.
- **poses** — the current world transform followed by the object's saved position history,
  each entry with its timestamp. The history exists so a reader can see the trajectory that
  led into the failure, not just the endpoint.
- **visual geometry** — the visual's bounding box, its centre and its radius. Empty when
  the object has no visual.
- **full** — identity, then properties, then poses, then geometry. The capped variant is
  the same with a banner, so a dump embedded in a longer message is findable.

**Notes** — "crow" is the project's word for an object that has been demoted to a cheap
distant update; the flag and its separate frame stamp are why the dump prints both.
