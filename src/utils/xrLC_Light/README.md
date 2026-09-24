# src/utils/xrLC_Light — what is left of the lighting compiler

Part of chapter 28 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

One file, one record, one format. The directory is named for the offline level-lighting
compiler — the tool that ran a radiosity solve over a level's geometry and produced its
baked lighting — and **that tool is not in this repository.** What remains is the single
header the renderer still needs: the on-disk shape of one baked light.

So this is not a module. It is a format declaration that ended up on the producer's side of
the fence, kept there because the producer owns the format even when the producer is gone.

## Where it sits

Outside the build order entirely. Nothing links it; the renderer
([chapter 18](../../Layers/xrRender/README.md)) includes the header directly across the
tree, which is a layering wart — a renderer reaching into a tools directory for a struct.
The right shape is for the level data format to be declared once where both a compiler and
a reader can see it, and a rebuild should move it.

## Load-bearing ideas, named once

**The baked light list is a bare array with no count.** A level's lighting chunk is a
sequence of fixed-size records, and the reader recovers how many there are by dividing the
chunk's length by the record's size, asserting the division is exact. That makes every byte
of the layout — field order, widths, padding — part of the format.

**The record carries three light kinds and the renderer reads one.** Directional, point and
secondary-bounce lights are all written; only point lights are turned into runtime lights.
The others are there because the format is shared with the compiler's own intermediate
passes.

**Several fields are precomputations and several are solver-only.** The squared range and
the falloff are redundant with the range and are stored because the compiler did the
arithmetic once over a few hundred thousand samples. The energy and the emitter triangle
are meaningful only inside the radiosity solve. A rebuild may recompute the first pair and
ignore the second, and must still write all of them, because the record's size is the
format.

**A rebuilder who needs to produce baked lighting has to write the compiler.** This page
and its twin are the only specification of what that compiler must emit.

## The files

| File | Role |
|---|---|
| [`R_light.h`](R_light.h.md) | The baked light record: its fields, their frozen widths, and which of them mean anything at run time |
