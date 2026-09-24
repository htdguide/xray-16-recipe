# src/xrGame/RGD5.h

> One named grenade type, existing only so that a class identifier in the spawn data has a class to instantiate.

**Needs** — [`Grenade.h`](Grenade.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a name and an inheritance edge

## Purpose

The shipped spawn data and configuration name a hand grenade whose class identifier maps
to this type. It adds no behaviour whatsoever to the generic grenade: every number — mass,
fuse time, blast radius, damage falloff, model, sounds — comes from its configuration
section.

It exists because the class-identifier-to-constructor table needs a distinct entry, and
because scripts identify this grenade by type. In a rebuild where entity classes are
looked up by section rather than by a compiled-in tag, this file disappears entirely and
the grenade becomes one more section of the generic grenade class. That is the honest
reading: the file is an artifact of a registration table, not a design decision.

## State

`Stateless.`

## `CRGD5`

**Contract** — a grenade. Default-constructible, destructible, nothing else. It declares a
script registration entry point exporting it as a type deriving from both the game object
facade and the explosive interface, so scripts can test for and cast to it; the
registration body lives with the other explosive registrations, not here.
