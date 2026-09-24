# src/xrServerEntities/pch_script.h

> The header aggregation for every translation unit in this directory that talks to the script layer.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`pch_script.cpp`](pch_script.cpp.md)
**Tier floor** — T4: a build artefact.

## Purpose

Names the script engine's export machinery and script-space header alongside the directory's
general aggregation, so that the roughly twenty script-export translation units here share
one compilation of a very expensive set of headers.

No decisions. A rebuild has nothing to carry over except the observation that **the script
exports are a separate compilation population from the records themselves** — the records
compile in the tools build with no script layer at all, and these files do not.

## State

`Stateless.`
