# src/xrCore/xr_trims.h

> Declares the separated-list and trimming vocabulary implemented in [`xr_trims.cpp`](xr_trims.cpp.md).

**Needs** — [`xr_trims.cpp`](xr_trims.cpp.md) · [`xrstring.h`](xrstring.h.md) · [`xr_token.h`](xr_token.h.md)
**Used by** — [`xrCore.h`](xrCore.h.md) · [`xr_ini.cpp`](xr_ini.cpp.md) · [`xr_trims.cpp`](xr_trims.cpp.md)
**Tier floor** — T3: string manipulation declarations.

## Purpose

Declares the list-splitting surface described in [`xr_trims.cpp`](xr_trims.cpp.md). Every function exists twice — once writing into a caller-supplied buffer, once into an owning string — and that duplication is incidental; a rebuild needs one of each.

## Exported units

- **Item count** — how many items a separated list holds; a doubled separator ends the list.
- **Get item** — the item at an index, trimmed by default, with a caller-supplied default when the index is out of range. A size-deducing form exists so the buffer length need not be passed.
- **Get items** — a half-open range of items, separators intact.
- **Set position** — the remainder of the list after skipping a number of items.
- **Copy value** — text up to the next separator.
- **Trim, trim left, trim right** — remove bytes at or below a threshold from the ends; the threshold defaults to the space character, which makes the default a whitespace-and-control trim.
- **Replace item, replace items** — substitute one item or a range.
- **Parse item** — resolve text, or an indexed item, against a name/number table; all-ones on a miss.
- **Change symbol** — substitute one character throughout.
- **Sequence to list** — split into owning, interned or duplicated strings, trimming each and dropping empties.
- **List to sequence** — join with commas.

The separator defaults to a comma everywhere, which is the format's convention.
