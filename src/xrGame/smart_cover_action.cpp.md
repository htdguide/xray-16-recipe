# src/xrGame/smart_cover_action.cpp

> Builds one loophole action from its authored table: the optional movement target, and the animation lists keyed by purpose.

**Needs** — [`smart_cover_action.h`](smart_cover_action.h.md) · [`smart_cover_detail.h`](smart_cover_detail.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: table parsing; no layout or device concern

## Purpose

Parses the smallest unit of a smart cover's description. The one structural decision here
is that an action's animations are a **map from purpose to a list of interchangeable
names**, not a single name: the same action at the same loophole can play any of several
clips, and the choice is made later, at the moment the creature performs it. That is what
keeps a squad of creatures using one cover from moving in lockstep.

## State

```text
RECORD action
  animations      : map<text, list<text>>   # purpose -> interchangeable clip names
  movement        : bool
  target_position : vector                  # cover-local; meaningful only when movement
```

**Invariants** — no list is empty and no list contains a name twice. Duplicates are
rejected at parse time because a later uniform pick over the list would then be biased,
and the bias would be invisible in the data.

## Construction

**Contract** — takes the action's authored table. Reads the movement flag if it is present
*and boolean* (anything else, including absent, means no movement); reads the target
position only when the movement flag was read at all. Then reads the animations table and
builds one list per purpose. Hard-fails if the animations table is absent or not a table —
an action with no animations is not an action.

```text
FUNCTION build_action(table) -> action
  m = table.movement
  IF m IS present AND m IS boolean
    movement = m
    IF table.position IS present
      target_position = table.position
  ELSE
    movement = false
  animations_table = REQUIRED table.animations
  FOR EACH (purpose, list) IN animations_table
    REQUIRE purpose IS text
    IF list IS NOT a table THEN SKIP          # tolerated; see Notes
    add_animation(purpose, list)
```

**Invariants** — the target position is left uninitialized when the movement flag is
absent. That is safe only because every reader checks the flag first; a rebuild should
make the pair one optional value and remove the hazard rather than reproduce it.

## `add_animation`

**Contract** — builds the list of clip names for one purpose. Non-string entries are
skipped, duplicates are a hard failure, and the list is sized from the entry count before
filling.

```text
FUNCTION add_animation(purpose, list)
  names = empty list sized to the entry count
  FOR EACH entry IN list
    IF entry IS NOT text THEN SKIP
    REQUIRE entry NOT already in names  ELSE FAIL WITH "duplicated animation"
    APPEND entry
  animations[purpose] = names
```

**Notes** —

- The skip-on-wrong-type arms throughout this file are paired with an assertion that the
  value is at least *present*. The intent readable from that pairing is: a hole in the
  table (an author's trailing comma, a conditionally-defined entry) is tolerated, but a
  value of the wrong kind is a mistake. A rebuild in a language with real sum types should
  express this as "the table may be sparse" and reject everything else.
- The original's skip arm for a non-string entry does not advance the iterator, so a
  malformed entry would spin. That is a latent defect, not a design decision; a rebuild
  must skip and advance.
