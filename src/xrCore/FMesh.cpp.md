# src/xrCore/FMesh.cpp

> Serializes the model file's authoring-provenance record.

**Needs** — [`FMesh.hpp`](FMesh.hpp.md) · [`FS.h`](FS.h.md)
**Used by** — [`FMesh.hpp`](FMesh.hpp.md)
**Tier floor** — T1: the timestamps are written as raw platform time values, whose width differs between builds.

## Purpose

The only member of the model format that needs a body rather than a declaration. Everything else about the container is described in [`FMesh.hpp`](FMesh.hpp.md).

## `ogf_desc.Load` / `ogf_desc.Save`

**Contract** — read or write the description chunk's seven fields in a fixed order: source path, build name, build time, create name, create time, modify name, modify time. Names are zero-terminated strings; times are raw platform timestamps.

```text
RECORD Description               # chunk 18 of a model file
  source_file : text (zero-terminated)
  build_name  : text (zero-terminated)
  build_time  : int              # SEE NOTE — width is the platform's time type
  create_name : text (zero-terminated)
  create_time : int
  modif_name  : text (zero-terminated)
  modif_time  : int
```

**Notes** — the timestamps are written with a raw whole-struct write of the platform's time type, which is 32 bits in the toolchain that produced the shipped data and 64 bits in a modern one. That makes this record's on-disk size build-dependent, which is a latent format bug: a 64-bit build writing this chunk produces a file a 32-bit build cannot read. It is harmless in practice only because nothing in the engine *reads* the field — it is shown in the tools and otherwise ignored, and a mis-parse of the chunk does not propagate. **A rebuild should fix the width to 64 bits** and accept that it cannot round-trip through the original exporter.
