# src/xrGame/CdkeyDecode/base32.h

> Declares the four text↔bytes conversions the product-key decoder is built from.

**Needs** — _(none)_
**Used by** — [`base32.c`](base32.c.md) · [`cdkeydecode.c`](cdkeydecode.c.md)
**Tier floor** — T3. Four functions over text and bytes; nothing here needs layout control.

## Purpose

Declares the surface implemented in [`base32.c`](base32.c.md). It is a separate unit from
the key decoder because the alphabet and the packing are generic: the decoder is the only
caller here, but the encoding is the vendor's, not the game's.

## Exported units

- `MakeBase32Pretty` — insert group separators into an encoded key for display.
- `CleanForBase32` — strip separators and upper-case, producing a string fit for decoding.
- `ConvertToBase32` — bytes → encoded text.
- `ConvertFromBase32` — encoded text → bytes.
