# src/xrGame/smart_cover_detail.cpp

> Reads typed fields out of an authored script table with a hard failure on anything malformed, and names the two pseudo-loopholes that represent standing outside the cover.

**Needs** — [`smart_cover_detail.h`](smart_cover_detail.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: dynamic-to-static field extraction

## Purpose

Smart covers are authored in the script language, as tables of tables. This file is the
boundary where that untyped data becomes engine values. Its one policy decision, applied
uniformly, is that **malformed authored data is a hard failure, not a default**: a missing
or wrongly-typed field fails with the field's name rather than silently yielding zero. A
cover with a silently-zeroed field of view would simply never be used and the author would
never learn why.

The exception to that policy is the small set of *optional* readers, which exist because
two fields were added after the format shipped — see the notes.

## State

```text
CONSTANT enter_loophole_id = "<__ENTER__>"
CONSTANT exit_loophole_id  = "<__EXIT__>"
```

**Invariants** — both names are bracketed in a way an author cannot produce by accident,
because they share a namespace with real loophole names and a collision would make the
transition graph nonsense.

## `parse_float`

**Contract** — two forms over one behaviour.

The **demanding** form reads a field, fails if it is absent or not a number, range-checks
it against an optional minimum and maximum, and returns it. The **optional** form writes
through an output and reports presence: absent or non-numeric yields no write and a
negative answer, but a present-but-out-of-range value still fails. That asymmetry is
deliberate — *missing* is a legal authoring state for these fields, *wrong* is not.

```text
FUNCTION parse_float(table, name, min = -inf, max = +inf) -> real
  REQUIRE table IS a table
  v = table[name]
  REQUIRE v IS present AND v IS a number   ELSE FAIL WITH "cannot read number value <name>"
  REQUIRE min <= v <= max                  ELSE FAIL WITH "invalid number value <name>"
  RETURN v
```

## `parse_string` / `parse_bool` / `parse_int` / `parse_table`

**Contract** — the same shape without the range check: read the named field, fail if it is
absent or of the wrong type, return it. `parse_table` writes through an output because a
table is not a value the caller can hold by return in the original; a rebuild should make
it a plain return like the others.

**Notes** — the type check is *exact*, not coercive. A field authored as the text "5"
where a number is wanted fails rather than converting. This is what stops an author's typo
from becoming a subtly wrong cover.

## `parse_fvector`

**Contract** — two forms, demanding and optional, reading a three-component position or
direction. Unlike the numeric readers, neither checks the *type* beyond presence: the
conversion from the script value is trusted to fail on its own if the value is not a
vector.

## `transform_vertex`

**Contract** — given a loophole name and a direction flag, returns the name unchanged if
it is non-empty, and otherwise substitutes the enter or the exit pseudo-loophole.

```text
FUNCTION transform_vertex(loophole_id, incoming) -> text
  IF loophole_id is non-empty  RETURN loophole_id
  RETURN enter_loophole_id IF incoming ELSE exit_loophole_id
```

**Invariants** — the direction flag is what disambiguates. The *same* empty name means
"from outside" when it appears as the source of a transition and "to outside" when it
appears as the destination, so the caller must know which end it is reading. Every reader
of a transition endpoint in the smart-cover loader passes this flag, and getting it
backwards produces a graph where nothing can be entered.

**Notes** — this is the trick that lets one graph express both movement *between*
loopholes and movement *into and out of* the cover. Rather than a separate entry/exit
table, the graph gets two extra vertices with reserved names, and entering the cover is
just a path starting at the enter vertex. Everything downstream — the path search in
[`smart_cover.cpp`](smart_cover.cpp.md), the animation planner — is uniform as a result.
An author writes an empty name and means "outside".

## `parse_vertex`

**Contract** — read a loophole name from a named field of a table (demanding), then pass it
through `transform_vertex` with the caller's direction flag.
