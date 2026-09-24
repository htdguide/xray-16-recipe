# src/xrCore/xr_trims.cpp

> The comma-separated-list vocabulary: how a configuration value that holds several items is split, counted, indexed, replaced and trimmed. Every tuple in the game's data is parsed through here.

**Needs** — [`xr_trims.h`](xr_trims.h.md) · [`xr_token.h`](xr_token.h.md) · [`xrstring.h`](xrstring.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md)
**Used by** — [`xr_trims.h`](xr_trims.h.md)
**Tier floor** — T3: string splitting over a separator. The in-place, caller-buffer forms are conveniences; the semantics are all that travel.

## Purpose

A configuration value is often a list: `wpn_ak74, wpn_abakan, wpn_lr300`, or a coordinate triple, or a section name followed by parameters. The configuration parser itself knows nothing about this — it hands back one string. This file is where that string becomes items, and the exact behaviour at the edges (empty items, trailing separators, what counts as whitespace) is what makes the shipped data parse the way it does.

Every function comes in two forms: one writing into a caller-supplied buffer, one into an owning string. The forms are behaviourally identical and their duplication is incidental.

## State

Stateless, with one exception noted under list-joining below.

## Trimming

**Contract** — Trimming removes, from one or both ends, every character whose byte value is **at or below** a given threshold, defaulting to the space character. Operates in place and returns the same string.

**Invariants** — The comparison is `<=`, not equality. Trimming with the default therefore removes every control character and every whitespace character in one pass, which is how carriage returns, tabs and line feeds vanish from parsed configuration lines. Trimming with an explicit character — the configuration parser trims with a double quote — removes that character *and everything below it*, which is a documented consequence, not a bug: trimming with a quote also strips surrounding whitespace.

```text
FUNCTION trim_left(s, threshold) -> text
  advance past every leading byte <= threshold, then shift the remainder down
FUNCTION trim_right(s, threshold) -> text
  walk back from the terminator past every byte <= threshold, terminate after the last kept byte
```

**Notes** — The right-hand trim starts at the terminator itself, which is byte zero and therefore always below the threshold, so it always steps back at least once. A string of nothing but trimmable bytes trims to empty. An empty string is left alone because the walk stops at the start.

## `item_count`

**Contract** — Counts the items in a separated list. An empty or absent string has zero items.

```text
FUNCTION item_count(s, separator) -> int
  count = 0
  scan forward for separators, counting each one
  # A doubled separator terminates the count early: "a,,b" counts 1, not 3.
  IF two separators are adjacent THEN stop
  IF anything follows the last separator THEN count = count + 1
  RETURN count
```

**Invariants** — Two consecutive separators end the list. This is the rule to carry: it makes a trailing `a,b,` count two items, and it makes `a,,b` count one. The shipped data relies on the first and never produces the second.

## `set_position` and `copy_value`

**Contract** — Positioning walks forward past a requested number of separators and returns the remainder, or nothing when the list runs out. Copying takes text up to the next separator, or all of it when there is none, into a caller buffer.

These two are the primitives; everything below is a composition of them.

## `get_item`

**Contract** — Extracts the item at an index into a buffer, trimming it by default. When the index is past the end, a caller-supplied default is copied instead (empty string by default). Returns the buffer.

**Notes** — The trimming is on by default and can be turned off, which matters for the few values where leading spaces are significant. The configuration vector reads do not use this path — they scan with a formatted parse — so a tuple's whitespace handling differs between "read as a vector" and "read as a list", and both behaviours are relied on.

## `get_items` — a range of items

**Contract** — Copies items from one index up to, but not including, another, *with their separators intact*, so the result is itself a valid list. Stops as soon as the end index is reached.

## `replace_item` / `replace_items`

**Contract** — Produces a new list in which the item at an index — or the half-open range of items — is replaced by given text. Everything outside the range is copied verbatim; the separators framing the range survive.

```text
FUNCTION replace_range(source, first, last, replacement, separator) -> text
  level = 0; emitted_replacement = false
  FOR EACH ch IN source
    IF level is within [first, last) THEN
      IF NOT emitted_replacement THEN emit replacement; emitted_replacement = true
      IF ch is separator THEN emit separator     # keep the framing
    ELSE
      emit ch
    IF ch is separator THEN level = level + 1
  RETURN emitted
```

**Notes** — The replacement is emitted once, at the first character of the range, and the range's own characters are then dropped. Separators inside the range are kept, so replacing two items with one item leaves a doubled separator — which, per the counting rule above, truncates the list. A rebuild should decide whether to keep that; the engine only ever replaces single items.

## `parse_item`

**Contract** — Resolves text, or the item at an index, against a name/number table by case-insensitive comparison, and returns the matching number. Returns all-ones (the 32-bit unsigned maximum) when nothing matches — which is the sentinel callers test against, and is distinct from the configuration parser's token read, which returns zero on a miss.

**Notes** — Two token-lookup functions with two different miss sentinels exist in this module ([`xr_token.cpp`](xr_token.cpp.md) returns -1, the configuration reader returns 0, this returns all-ones). That is an inconsistency, not a design; a rebuild should pick one and say so.

## `sequence_to_list` / `list_to_sequence`

**Contract** — Splitting produces one entry per item, each trimmed, **with empty entries dropped**. Three variants exist, one per string representation; the owning-string variants clear the destination first, the raw-pointer variant appends duplicated strings the caller must free.

Joining is the inverse: items separated by commas with no spaces.

**Invariants** — Empty items are dropped on split but a joined list never produces them, so the pair round-trips for every list the engine actually builds.

**Notes** — The joining function for owning strings returns a reference to a **function-local static** buffer, which the source itself flags as able to crash at process exit and which makes the function unusable from two threads at once. It is a defect to be fixed in a rebuild, not a decision: return a value.

## `change_symbol`

**Contract** — Replaces every occurrence of one character with another, in place. Used to normalize path separators and to turn separators into something a formatted parse will not stop on.
