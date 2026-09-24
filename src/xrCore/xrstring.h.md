# src/xrCore/xrstring.h

> Declares the interned-string record, the global interner and the handle type implemented in [`xrstring.cpp`](xrstring.cpp.md), plus the ASCII string helpers the whole engine uses.

**Needs** — [`xrstring.cpp`](xrstring.cpp.md) · [`xr_types.h`](xr_types.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`object_comparer.h`](../Common/object_comparer.h.md) · [`object_loader.h`](../Common/object_loader.h.md) · [`object_saver.h`](../Common/object_saver.h.md) · [`predicates.h`](../xrCommon/predicates.h.md) · [`xr_string.h`](../xrCommon/xr_string.h.md) · [`xr_unordered_map.h`](../xrCommon/xr_unordered_map.h.md) · [`Bone.hpp`](Animation/Bone.hpp.md) · [`SkeletonMotions.hpp`](Animation/SkeletonMotions.hpp.md) · [`xr_dsa.cpp`](Crypto/xr_dsa.cpp.md) · [`xr_dsa.h`](Crypto/xr_dsa.h.md) · [`FMesh.hpp`](FMesh.hpp.md) · [`FS.h`](FS.h.md) · [`FileSystem.cpp`](FileSystem.cpp.md) · [`LocatorAPI_auth.cpp`](LocatorAPI_auth.cpp.md) · _and 25 more_
**Tier floor** — T1: it declares a record whose characters live in the same allocation as its header, at a fixed offset, and whose identity is its address.

## Purpose

Declares the interned string record, the interner's surface and the handle type — all described in [`xrstring.cpp`](xrstring.cpp.md). It additionally carries the small set of free functions that make a handle behave like text everywhere else in the engine, which is why nearly every file includes it.

## Exported units

- **`InternedString` record** — reference count, length, checksum, chain link, characters. Four-byte packed so the character array starts at a known offset.
- **`Interner`** — the process-wide table: intern, sweep unreferenced, verify, dump, report savings.
- **The global interner handle** — created during memory initialization, before the filesystem, and destroyed after it.
- **`shared_str` handle** — the reference-counted pointer to a record: construct from text or another handle, move, assign, release, index a character, ask for text, length, emptiness, swap, and identity comparison.
- **Comparison and ordering** — equality and ordering on record identity, never on text. Comparison against a null literal is made unavailable so callers spell emptiness explicitly.
- **Hash** — the record's stored checksum.
- **Formatted assignment** — format into a scratch buffer (4096 bytes) and intern the result.
- **`xr_strcmp` family** — text comparison across every mix of handle and raw text; the handle/handle form short-circuits on identity.
- **`xr_strlwr` family** — ASCII, locale-independent lowercasing of raw text in place, and of a handle by re-interning.

**Notes** — The lowercase helper is ASCII-only *by requirement*, not by oversight: game text ships in several single-byte codepages and a locale-aware fold changes which files a path lookup matches.
