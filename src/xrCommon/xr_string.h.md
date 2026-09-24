# src/xrCommon/xr_string.h

> The engine's *mutable* text buffer — the one you edit — as distinct from the interned, shared, immutable text the rest of the engine passes around.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md)
**Used by** — [`object_loader.h`](../Common/object_loader.h.md) · [`object_saver.h`](../Common/object_saver.h.md) · [`StackTrace.h`](../xrCore/Debug/StackTrace.h.md) · [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`log.h`](../xrCore/log.h.md) · [`net_utils.h`](../xrCore/net_utils.h.md) · [`os_clipboard.cpp`](../xrCore/os_clipboard.cpp.md) · [`string_concatenations_inline.h`](../xrCore/string_concatenations_inline.h.md) · [`xrDebug.h`](../xrCore/xrDebug.h.md) · [`xr_trims.cpp`](../xrCore/xr_trims.cpp.md) · [`UIGameCustom.h`](../xrGame/UIGameCustom.h.md)
**Tier floor** — T2: the text is bytes in a chosen encoding, not code points, and callers hand its buffer address to formatting and filesystem calls.

## Purpose

The engine has two text types and the split is the important thing on this page.

- **Interned text** ([`xrCore/xrstring.h`](../xrCore/xrstring.h.md)) is what the engine
  *stores and compares*: immutable, reference-counted, de-duplicated in a process-wide
  intern table, compared in one pointer comparison, hashed in one field read. Almost every
  name in the engine — resource paths, section names, bone names, class identifiers — is
  this type.
- **Mutable text** (this file) is what the engine *builds*: a growable byte buffer that
  supports append, search, replace and case change. It exists because you cannot edit an
  interned string, only replace it.

The intended flow is build-then-intern: assemble a path or a message in a mutable buffer,
then hand it to the intern table and keep the interned result. A rebuild with one string
type still needs both *behaviours*, and needs to know that switching a stored field from
interned to mutable multiplies its memory by the duplication factor of the game data,
which for level resource names is large.

## State

```text
RECORD MutableText
  bytes  : bytes          # not code points; see encoding note below
  # Storage is obtained from the engine allocation policy, like every container
  # in this chapter. Short values are NOT guaranteed to avoid allocation - the
  # engine assumes nothing here and neither should a rebuild.
```

## `xr_string`

**Contract** — a growable, mutable sequence of bytes with the usual text operations:
length, indexed access, concatenation, substring search returning a position or a
not-found marker, substring replacement by position and length, and access to a
null-terminated address for handing to lower-level interfaces.

**Invariants**

```text
# Content is BYTES in a single-byte codepage, not Unicode text.
#   Game text is authored in one of several codepages depending on the
#   localization. Length is a byte count. Indexing is byte indexing. A rebuild
#   whose native string is Unicode must decide where the codepage-to-Unicode
#   translation happens - the engine does it on load, at the XML text-table
#   reader - and must not let its string type silently re-encode data on the way
#   to a file or a network packet.
```

## `xr_strlwr` — fold to lower case, in place

**Contract** — replaces every byte of the buffer with its lower-case form, in place, same
length. Returns nothing; the argument is modified.

**Invariants**

```text
# The fold is per-byte and ASCII-only, matching the ordering in predicates.h.
#   Same reason: the set of files a level matches must not depend on the user's
#   locale. Bytes outside the unaccented Latin alphabet pass through unchanged,
#   which means localized text is NOT folded - and that is correct, because
#   nothing case-folds display text, only identifiers.
```

## `xr_substrreplace` — replace every occurrence

**Contract** — takes a source text, a pattern and a replacement; returns a new text in
which every occurrence of the pattern has been replaced. Neither argument is modified. Used
on paths and on configuration values, never in a per-frame path.

```text
FUNCTION replace_all(source, pattern, replacement) -> text
  result = copy of source
  LOOP
    at = first position of pattern in result    # search restarts at the front
    IF at is none
      RETURN result
    result = result with pattern at `at` replaced by replacement
```

**Invariants**

```text
# The search restarts from the front after every replacement.
#   Consequence: if the replacement text CONTAINS the pattern, this never
#   terminates. Nothing in the engine passes such a pair, and nothing checks.
#   A rebuild should either scan forward past each replacement (which also makes
#   it linear rather than quadratic) or reject the self-containing case loudly.
#   Scanning forward changes behaviour only for inputs that currently hang, so
#   it is a safe fix.

# An empty pattern is not handled.
#   It is found at position zero forever. Same class of defect, same fix.
```

## Hashing

**Contract** — mutable text can be used as a key in a hashed table; its hash is taken over
the bytes up to the first zero byte.

**Notes** — the hash reads the buffer as a null-terminated address rather than as an
address-and-length pair, so a value containing an embedded zero byte hashes only its
prefix while comparing as its whole content. Hash and equality then disagree, and a hashed
table keyed by such a value loses entries. No caller stores embedded zeroes, which is why
this has never been hit; a rebuild should hash the length-delimited content and the
question disappears.

This hash is also defined only on the non-Windows builds, because the platform's own
standard library already supplied one there. That conditional is pure C++ ecosystem
accident and survives as nothing.
