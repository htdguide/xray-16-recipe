# src/editors/xrWeatherEngine/pch.cpp

> Nothing. It exists so the build has one translation unit to compile the prelude from.

**Needs** — [`pch.hpp`](pch.hpp.md)
**Used by** — reached through its declarations in [`pch.hpp`](pch.hpp.md); callers name that, not this file.
**Tier floor** — T4: a build artifact with no runtime content.

## Purpose

Incidental to C++'s precompiled-header mechanism. A rebuild deletes it.

## State

`Stateless.`
