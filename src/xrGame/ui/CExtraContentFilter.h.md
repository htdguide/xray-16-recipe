# src/xrGame/ui/CExtraContentFilter.h

> Declares the gate that hides bonus-edition content from players who did not buy it.

**Needs** — [`CExtraContentFilter.cpp`](CExtraContentFilter.cpp.md)
**Used by** — [`CExtraContentFilter.cpp`](CExtraContentFilter.cpp.md)
**Tier floor** — T3: a name lookup over a small table

## Purpose

Declares the surface implemented in [`CExtraContentFilter.cpp`](CExtraContentFilter.cpp.md).

## `CExtraContentFilter`

The filter object. Built once, it reads the pack table out of configuration and asks the
per-machine settings store whether each pack is unlocked.

- **construct** — load the pack table and resolve each pack's unlocked flag.
- **`IsDataEnabled(name)`** — is this named piece of content available to this player.
