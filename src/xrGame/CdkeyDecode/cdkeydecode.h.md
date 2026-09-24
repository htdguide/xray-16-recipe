# src/xrGame/CdkeyDecode/cdkeydecode.h

> Declares the two questions the game may ask of a product key: what is hidden inside it,
> and does it look like one this publisher issued.

**Needs** — _(none)_
**Used by** — [`MainMenu.cpp`](../MainMenu.cpp.md)
**Tier floor** — T3. Two functions over text.

## Purpose

Declares the surface implemented in [`cdkeydecode.c`](cdkeydecode.c.md). It is the whole
public face of this directory — [`base32`](base32.h.md) is an implementation detail behind
it and no other part of the engine includes it.

## Exported units

- `DecodeKeyData` — recovers the payload bytes a key carries, or reports zero bytes on any
  malformed key.
- `VerifyClientCheck` — decides whether a key's embedded checksum agrees with a
  publisher-supplied salt. A local sanity test only; it cannot tell whether the key was
  ever issued.
