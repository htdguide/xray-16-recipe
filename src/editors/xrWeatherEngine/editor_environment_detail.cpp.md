# src/editors/xrWeatherEngine/editor_environment_detail.cpp

> Sort names the way a person reads them, and turn a virtual path into one a file dialog can open.

**Needs** — [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md)
**Tier floor** — T1: the ordering is delegated to a platform text-comparison routine, over a text encoding it converts to first.

## Purpose

Two unrelated helpers that share a file because both are one-liners used everywhere. The
first is the ordering rule for every browsable name list in the weather model; the second
is the bridge between the engine's virtual filesystem and the native file dialogs the
grid's browse buttons open.

## State

`Stateless.`

## The natural-order comparator

**Contract** — orders two names as a person would: runs of digits compare by numeric
value, not character by character, so `sun_9` precedes `sun_10`. Offered for both plain
text and interned strings, with identical behaviour. Does not allocate on the heap; does
not block.

```text
FUNCTION natural_less(a : text, b : text) -> bool
  RETURN platform_natural_compare(widen(a), widen(b)) < 0
```

**Notes** — Delegating to the platform is why this file is the module's only non-portable
one, and the delegation is not free: the platform routine wants wide characters, so both
names are converted first, on the call stack, sized from their own lengths.

What must survive a rebuild is the *rule*, not the delegation: **every list the author
browses is in natural order**, because these lists are dense with numbered names —
`thunderbolt_unique_id_0` through `thunderbolt_unique_id_12`, sky textures numbered by
time of day — and plain text order scatters them. A rebuild implements the comparison
itself and drops the encoding conversion with it.

## `real_path`

**Contract** — resolves a virtual filesystem folder alias and a relative path into a
concrete location. Used only to give a native file dialog a starting folder.

```text
FUNCTION real_path(folder_alias : text, path : text) -> text
  RETURN filesystem.resolve(folder_alias, path)
```

**Notes** — Every browsable-file row in this module calls it with an empty relative path
and one of three aliases: the texture folder, the sound folder, or the mesh folder. The
alias is resolved at the moment the row is built, not when the dialog opens, so an author
who remounts the game data mid-session keeps the old starting folder until the grid is
rebuilt.
