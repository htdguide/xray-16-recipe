# src/utils/xrMiscMath/pch.cpp

> Gives the shared prologue a compilation unit of its own.

**Needs** — [`pch.hpp`](pch.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — None: a build artifact with no runtime behaviour

## Purpose

Exists so the toolchain has somewhere to compile the shared prologue from. It contributes
no code. Nothing about it survives into a rebuild.

## State

`Stateless.`

## Exported units

None.
