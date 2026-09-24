# src/editors/xrWeatherEditor/property_converter_integer_enum.cpp

> Renders a whole number as the label of its declared choice — the ordinary enumeration case.

**Needs** — [`property_converter_integer_enum.hpp`](property_converter_integer_enum.hpp.md) · [`property_integer_enum_value.hpp`](property_integer_enum_value.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_converter_integer_enum.hpp`](property_converter_integer_enum.hpp.md)
**Tier floor** — T3: a linear search and a render.

## Purpose

An integer chosen from a named set: a sound channel's mode, a thunderbolt's variant, a flare's blending rule. This is the shape an enumeration actually wants, and it is the same file as [`property_converter_float_enum`](property_converter_float_enum.cpp.md) with the value type changed.

## State

`Stateless.`

## The dropdown

**Contract** — the binding's (integer, label) pairs, offered exclusively.

## Rendering

**Contract** — text passes through; a pair renders as its label; a bare integer is matched against the pairs and renders as the matching label, falling back to the first pair's label when nothing matches.

**Invariants** — the match is exact integer equality, which is sound here in a way it is not for the real-valued sibling: an integer that has been through arithmetic still compares exactly.

**Notes** — the same fallback lie as the real-valued case: an out-of-set stored value displays as the first choice, and touching the row commits it. Here it is more likely to bite, because an integer field's set can change between engine versions while the authored data does not — a keyframe written by a newer build and opened in an older one silently rewrites to the first choice.

A rebuild should render an unmatched integer as itself.

## Notes

This file and [`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md) are identical but for the value type they compare. A rebuild with a generic renderer over a choice list writes one.
