# src/xrGame/ExoOutfit.cpp

> The powered exoskeleton suit: the base outfit under its own class identifier, with every difference expressed in configuration.

**Needs** — [`ExoOutfit.h`](ExoOutfit.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identity and nothing else

## Purpose

A distinct class identifier for the exoskeleton, with no behaviour of its own. Everything
that makes an exoskeleton different — the mass it adds, the stamina it costs, the
protection it grants, whether it permits a helmet — comes from its configuration section,
read by the base outfit. The class exists only because the shipped spawn records and
inventory data name it, and because the script layer and some mission logic test for the
type.

## State

`Stateless.`

## `CExoOutfit`

**Contract** — construction and destruction, both empty; every operation is the base
outfit's. See [`CustomOutfit.cpp`](CustomOutfit.cpp.md) for what an outfit actually does.
