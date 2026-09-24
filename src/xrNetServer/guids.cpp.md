# src/xrNetServer/guids.cpp

> A four-line compilation unit whose entire job is to make the transport library's identifier
> constants exist somewhere in the program.

**Needs** — [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: build plumbing for one specific vendor library on one platform.

## Purpose

The vendor transport library publishes its interface and provider identifiers as declarations
only; one translation unit in the program must ask for them to be defined, or nothing links.
This file is that unit. On platforms without the library it compiles to nothing at all.

It is the purest example in the module of a file that is *entirely* incidental. The decision
it encodes — "somebody has to materialize the vendor's constants" — evaporates the moment the
vendor does, which is the whole argument for treating the transport as a
[given seam](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).

## State

Stateless.

## Notes

The same request is also made in [`stdafx.cpp`](stdafx.cpp.md). One of the two files is
redundant and has been since before this repository's history begins.
