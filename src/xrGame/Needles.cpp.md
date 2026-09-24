# src/xrGame/Needles.cpp

> An artefact class that exists only as a name in the class-identifier table: it inherits everything and overrides nothing.

**Needs** — [`Needles.h`](Needles.h.md) · [`Artefact.h`](Artefact.h.md)
**Used by** — [`Needles.h`](Needles.h.md)
**Tier floor** — T3: a named leaf of the entity class hierarchy with no behaviour of its own

## Purpose

The class-identifier table in the shipped game data names more artefacts than the engine
implements distinctly. The needles artefact is one of those names: every property it has —
which damage types it resists, what it does to the carrier's health and radiation, its price
and its visual — comes from its configuration section, not from code. The class exists so that
a spawn record carrying its identifier instantiates something.

A rebuild that drives its class-identifier table from data will get this file for free and need
not write it at all.

## State

`Stateless.`

## `CNeedles`

**Contract** — constructs and destroys with no work. All behaviour is the base artefact's.

**Notes** — the file's header comment names a different artefact, carried over from whichever
file it was copied from. Nothing follows from it.
