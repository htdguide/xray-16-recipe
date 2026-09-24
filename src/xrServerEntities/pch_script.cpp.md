# src/xrServerEntities/pch_script.cpp

> The compilation anchor for the script-facing header aggregation.

**Needs** — [`pch_script.h`](pch_script.h.md)
**Used by** — reached through its declarations in [`pch_script.h`](pch_script.h.md); callers name that, not this file.
**Tier floor** — T4: a build artefact.

## Purpose

Exists so the aggregated headers have a translation unit to be compiled into. It also
silences one compiler diagnostic about symbol-name length, which the binding layer's deeply
nested types provoke in enormous quantity. Nothing survives into a rebuild.

## State

`Stateless.`
