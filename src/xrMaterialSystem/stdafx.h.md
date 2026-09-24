# src/xrMaterialSystem/stdafx.h

> The module's compile-time prelude: the engine prelude, the core library, and this module's own public header.

**Needs** — [`Common/Common.hpp`](../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [`GameMtlLib.h`](GameMtlLib.h.md)
**Used by** — [`stdafx.cpp`](stdafx.cpp.md)
**Tier floor** — T4: it names three things in an order and computes nothing.

## Purpose

A build-time aggregation that pre-compiles the three headers every source in this module
opens with, so that they are parsed once rather than three times. It carries no decision
and no program meaning.

## State

Stateless.

## Notes

A rebuild in a language with modules has no equivalent and should not invent one. The one
fact worth carrying across is the dependency it states: this module rests on the engine
prelude and the core library and on nothing else — not on the game, not on physics, not on
the renderer's implementation. The renderer and the sound device are reached only through
their interfaces, which is what lets the material library sit as early in the build order
as it does.
