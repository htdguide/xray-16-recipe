# src/utils/xrCompress — the archive packer

Part of chapter 28 of [the build order](../../../SYSTEM-REQUIREMENTS.md#7-build-order).

## What this module is responsible for

Writing the frozen archive format that the engine reads. The virtual filesystem chapter
([`src/xrCore`](../../xrCore/README.md)) describes how an archive is *mounted* — how the
directory is found, how an entry is located, how a payload is decompressed in place. This
directory is the other half, and it is the only place in the repository that describes the
**write** side. The two must agree byte for byte or no shipped data loads, so read this
chapter as the authoritative statement of the format's producer and
[`LocatorAPI.cpp`](../../xrCore/LocatorAPI.cpp.md) as its consumer.

It also carries a second, unrelated program in the same executable: a differencing pass
that produces the file set for a patch. The two share nothing but the binary.

## Where it sits

At the very end. It rests on the core layer alone — the virtual filesystem, the
configuration parser, the checksum and the compressor — and nothing rests on it. It is not
linked into the game; it is run by whoever builds the data.

The important consequence of resting on the *same* core layer the engine uses is that path
normalization, the configuration parser and the checksum are the identical
implementations on both sides of the format. A rebuild that reimplements the packer
against a different set of primitives has to reproduce those three exactly, because all
three leak into the bytes.

## Load-bearing ideas, named once

**The archive is a chunked container, and the directory is at the end.** A volume is a
sequence of tagged chunks: an optional mount-configuration chunk, one data blob, and a
compressed directory. The directory is written last because its entries carry the offsets
of payloads that are not known until those payloads have been placed.

**An entry is sixteen bytes plus a name.** Four 32-bit fields — real size, stored size,
checksum, offset — and a 16-bit length that counts the fixed fields plus the name and
excludes itself. Names are not terminated; the reader recovers a name's length by
subtracting sixteen. Every one of these widths is the format.

**Compression is signalled by equality, not by a flag.** An entry whose stored size equals
its real size is stored verbatim; any other value means the payload is compressed. There
is no bit anywhere that says which. This is the single most consequential frozen decision
in the format and it constrains the packer: a payload that *compresses to exactly its own
size* must be stored, not compressed, or the reader will hand the compressed bytes back as
if they were the file.

**The checksum is always over the uncompressed payload.** Including for stored entries,
where it doubles as a checksum of what is literally on disk.

**Offsets are absolute within the volume and 32-bit.** That is why volumes are capped just
under two gigabytes, and why a pack that would exceed the cap splits into numbered volumes
rather than failing. The cap is not a tuning parameter; it is the width of the field.

**Identical payloads are stored once.** Game data carries the same bytes under many names,
so the packer keys a table by payload size, compares candidates by checksum and then by
content, and points a duplicate entry at the bytes already placed. Deduplication is
invisible to the reader — two entries simply share an offset.

**Some file types are deliberately never compressed**, because the engine reads them
through a path that would pay a decompression cost on every access, and some compress so
badly that storing them is smaller. The list is a property of the data, not of the format.

**Two archive variants exist**, a base-game one and a patch one, differing in which chunks
are emitted rather than in how payloads are stored. The patch variant carries a
mount-configuration chunk copied verbatim from a file the operator supplies, which is how
a patch declares the filesystem roots it wants mounted.

**The differencing pass never expresses a deletion.** It copies out every file of a new
build that the old build does not already have identically — by name, size, checksum and
full content, each test independently switchable. A patch built this way can add and
replace, and cannot remove.

## The files

| File | Role |
|---|---|
| [`xrCompress.cpp`](xrCompress.cpp.md) | The packer: the container's layout, the store-or-compress decision, deduplication, volume splitting, the directory |
| [`xrCompress.h`](xrCompress.h.md) | The packer's job description and its two entry points |
| [`main.cpp`](main.cpp.md) | The command line: option names, the single-folder mount, the pack/difference dispatch |
| [`xrCompressDifference.cpp`](xrCompressDifference.cpp.md) | The patch pass: what "unchanged" means and how the changed set is copied out |
| [`StdAfx.h`](StdAfx.h.md) | Build-time header aggregation; no decisions |

One further file in the directory is build description — a package reference list for the
build tool — and carries no decisions.
