# src/xrGame/BlackDrops.cpp

> An artefact with no behaviour of its own beyond what its configuration section gives it.

**Needs** — [`BlackDrops.h`](BlackDrops.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a named subclass with no added behaviour

## Purpose

A distinct artefact class that adds nothing to the base artefact. It exists so that the
shipped data's class-identifier table has a tag for it, and so that scripts can test for
this artefact's type by class rather than by section name.

That is a real requirement — the class-identifier values ship in the game data and are
frozen — but it means the file is empty of decisions. A rebuild whose factory can map a
class identifier to "base artefact with this section" needs no class here at all.

## State

`Stateless.` Everything this artefact is comes from its section, read by the base class.

## `CBlackDrops`

**Contract** — a base artefact under a different name. Its load path delegates entirely.
