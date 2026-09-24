# src/editors/xrWeatherEditor/pch.cpp

> The translation unit that exists so the shared preamble has something to compile into.

**Needs** — [`pch.hpp`](pch.hpp.md)
**Used by** — reached through its declarations in [`pch.hpp`](pch.hpp.md); callers name that, not this file.
**Tier floor** — T4: a build artifact with no content.

## Purpose

It includes [`pch.hpp`](pch.hpp.md) and nothing else. Its only reason to exist is that the toolchain's shared-preamble mechanism needs one translation unit to build the preamble from.

## State

`Stateless.`

## Notes

Entirely incidental. A rebuild in any language with a module system has no analogue and needs none.
