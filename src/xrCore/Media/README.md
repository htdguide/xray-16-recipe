# src/xrCore/Media — the one in-memory raster image

Part of chapter 6, [`src/xrCore`](../README.md). Three files.

## What this module is responsible for

A raster image held in memory, and the two file forms it can be written to or read from: an
uncompressed true-colour container, and a lossy one. That is the whole scope, and the scope
is small for a reason — **this is not the texture pipeline.** Game textures are
block-compressed, mip-chained, and handed to the graphics device without ever being decoded
([Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)). Nothing in this
directory touches them.

What it serves is the handful of places the engine needs actual pixels: the screenshot the
player takes, the loading screens stored in a lossy container, and the thumbnail a save game
carries.

## Where it sits

It rests on the chunked container's writer surface ([`../FS.h`](../FS.h.md)) for output and
on an external lossy codec, which is optional — the feature compiles out and the twin says
what happens then. It is consumed by chapter 13's screenshot path and chapter 23's save
system.

## The load-bearing ideas

**The lossy codec is a seam with a hard edge.** It reports fatal errors by longjmp — it does
not return them — so the bridge must convert that into an ordinary failure before it reaches
engine code. [`ImageJPEG.cpp`](ImageJPEG.cpp.md) is mostly that conversion, and a rebuild
whose codec reports errors normally deletes most of the file.

**Output goes through the engine's writer, not through the codec's own file handling.** Every
byte the codec produces is routed into a writer from [`../FS.h`](../FS.h.md), which is what
lets a screenshot be written into a logical root rather than a physical path, and what lets a
save thumbnail be embedded in a chunk rather than written to its own file.

**The whole subsystem is optional.** Built without the codec, the lossy path is absent and
the uncompressed one remains. Nothing in the engine's correctness depends on it.

## The twins

| File | Role |
|---|---|
| [`Image.hpp`](Image.hpp.md) | Declares the raster: pixels, a format tag, and the two containers it can round-trip through. |
| [`Image.cpp`](Image.cpp.md) | **Holding a raster and writing it out uncompressed.** Substantive. |
| [`ImageJPEG.cpp`](ImageJPEG.cpp.md) | The lossy bridge: route the codec through an engine writer, turn its fatal errors into a recoverable failure, and make the feature optional. |
