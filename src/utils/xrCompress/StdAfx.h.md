# src/utils/xrCompress/StdAfx.h

> The packer's build-time header aggregation — it names the core layer and the compressor, and decides nothing.

**Needs** — [`Common/Common.hpp`](../../Common/Common.hpp.md) · [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)

**Used by** — [`main.cpp`](main.cpp.md) · [`xrCompress.cpp`](xrCompress.cpp.md) · [`xrCompressDifference.cpp`](xrCompressDifference.cpp.md)

**Tier floor** — T4: it is a list of names, not a computation.

## Purpose

Declares the packer's world: the platform vocabulary, the core layer (virtual filesystem,
configuration parser, strings, logging) and the byte-oriented compressor whose output
format is frozen by every shipped archive. Nothing here survives into a rebuild as itself.

The one fact worth carrying across: the packer links against the **same** core layer the
engine does, so the path normalization, the configuration parser and the checksum used to
*write* an archive are the identical implementations used to *read* one. That is not a
convenience — it is why the two sides cannot drift.
