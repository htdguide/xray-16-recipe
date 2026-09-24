# src/xrGame/ScientificOutfit.cpp

> The scientist's protective suit: a named outfit type with no behaviour of its own.

**Needs** — [`ScientificOutfit.h`](ScientificOutfit.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identifier with a name

## Purpose

An outfit is entirely data: per-damage-type protection multipliers, carrying-weight bonus,
the headgear and body models it swaps in, the sounds it makes, its condition decay and its
repair cost. All of it is read by the generic outfit from the configuration section, so a
specific outfit class adds nothing but an identity.

This one exists because the shipped class-identifier table names it. A rebuild that
resolves classes by section deletes it.

## State

`Stateless.`

## `CScientificOutfit`

**Contract** — a generic outfit. Construction and destruction do nothing.
