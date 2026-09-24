# src/xrGame/MilitaryOutfit.cpp

> A protective suit class that exists only so the class-identifier table has something to name: it inherits everything and overrides nothing.

**Needs** — [`MilitaryOutfit.h`](MilitaryOutfit.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md)
**Used by** — [`MilitaryOutfit.h`](MilitaryOutfit.h.md)
**Tier floor** — T3: a named leaf of the entity class hierarchy with no behaviour of its own

## Purpose

The shipped game data names more outfit kinds than the engine implements distinctly. The
military suit is one of those names: every difference between it and any other suit —
protection per damage type, weight, carrying capacity, the visual — comes from its
configuration section, not from code. The class exists so a spawn record carrying its class
identifier instantiates something.

A rebuild that drives its class-identifier table from data will get this file for free and
need not write it at all.

## State

`Stateless.`

## `CMilitaryOutfit`

**Contract** — constructs and destroys with no work. All behaviour is the custom outfit's.
