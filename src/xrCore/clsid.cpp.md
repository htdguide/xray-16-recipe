# src/xrCore/clsid.cpp

> Class identifiers are eight-character names packed big-endian into a 64-bit integer — the key every spawn record uses to say what kind of entity it is.

**Needs** — [`clsid.h`](clsid.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrDebug.h`](xrDebug.h.md)
**Used by** — [`clsid.h`](clsid.h.md)
**Tier floor** — T1: it is a frozen packing of characters into an integer whose value appears in shipped spawn files and configuration text.

## Purpose

Every entity class in the game is named by an eight-character identifier — `AI_STL_S`, `W_AK74`, `SCRPTCAR` — written as text in configuration and carried as a number in the level's spawn file and in the factory's registration table. This file is the bijection between the two forms. It is **frozen**: the numbers are in shipped data.

## State

Stateless.

## `text_to_class_id`

**Contract** — Takes up to eight characters and returns the packed identifier. Longer input fails an assertion (in debug builds) and is truncated to eight. Shorter input is **padded on the right with spaces** to exactly eight. Allocation-free, pure.

```text
FUNCTION text_to_class_id(text) -> int (64-bit)
  REQUIRE length(text) <= 8
  eight = first 8 characters of text, right-padded with spaces
  # Big-endian packing: the FIRST character occupies the HIGHEST byte.
  # Ordering identifiers as integers therefore orders them as text,
  # which several registration tables rely on.
  result = 0
  FOR i FROM 0 TO 7
    result = result OR (byte_value(eight[i]) SHIFTED LEFT (8 * (7 - i)))
  RETURN result
```

**Invariants** — Space padding, not zero padding. `W_AK74` and `W_AK74\0\0` are **not** the same identifier; `W_AK74  ` is what the shipped data means. A rebuild that zero-pads produces identifiers that match nothing.

## `class_id_to_text`

**Contract** — The inverse: unpacks eight bytes, highest first, into a caller-supplied buffer of at least nine bytes, terminated. Trailing spaces are **not** stripped — the round trip is exact.

## Notes

The same packing is available as a compile-time expression over a character array or eight separate characters, so identifiers written in source become integer constants with no runtime work and can be used as case labels. That is the form every class-registration table uses.

The 64-bit width matters: the original engine used a 32-bit identifier built from four characters in some tables and eight-character text in others, and this codebase settled on eight characters everywhere. A rebuild reading original data must use the eight-character form.
