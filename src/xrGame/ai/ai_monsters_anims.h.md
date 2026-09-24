# src/xrGame/ai/ai_monsters_anims.h

> Loads a creature's animation set by *naming convention*, so a new creature's motions can be added in data without touching code.

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md) · [`ai_debug.h`](../ai_debug.h.md) · [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md)
**Used by** — [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md) · [`stalker_animation_pair.h`](../stalker_animation_pair.h.md) · [`stalker_animation_state.h`](../stalker_animation_state.h.md)
**Tier floor** — T3: string composition and a name lookup; nothing here touches layout or the clock

## Purpose

A creature's animation data is authored as families of motions with numbered variants —
`death_0`, `death_1`, `death_2` — and the engine picks one at random each time so that ten
dogs dying do not die identically. This file is the loader for that convention: given a
model and a base name, it collects every motion whose name is that base with a numeric or
a suffix appended, and hands back a list the behaviour code indexes or samples.

It exists as a separate file because three different shapes of the same idea are needed,
and every creature uses at least one of them. It is a *convention*, not a format: the only
thing frozen is that the shipped models name their motions this way.

## State

```text
RECORD MotionList
  motions : list<motion handle>     # in suffix order; index is meaningful for suffix-driven lists
```

**Invariants** — a lookup that finds no motion yields an empty handle, which every consumer
must treat as "this creature does not have this animation" rather than as an error. Only the
per-creature code knows which of its animations are mandatory.

## `numbered motion list`

**Contract** — given a model and a base name, appends the decimal integers `0`, `1`, `2`, …
to the base and collects every motion that resolves. A number that does not resolve as a
*cycle* is retried as an *effect* motion (the two live in separate name spaces in the model
format and some creatures author one-shot hits as effects). Scanning stops at the first
gap *at or after index 10*; gaps below 10 are skipped, not fatal.

```text
FUNCTION load_numbered(model, base) -> MotionList
  out = empty
  i = 0
  LOOP
    name = base + decimal(i)
    m = model.find_cycle(name)
    IF m is none THEN m = model.find_effect(name)
    IF m is not none THEN
      out.append(m)
    ELSE IF i >= 10 THEN
      BREAK                # a gap only ends the scan once ten indices have been offered
    i = i + 1
  RETURN out
```

**Notes** — the "keep going until index 10" rule is what makes the shipped data loadable: a
creature may author `attack_0` and `attack_3` with nothing between, and a scanner that
stopped at the first miss would silently lose the later variants. Ten is the point past
which the loader assumes the family is genuinely finished. Nothing in the source derives
ten; it is a tolerance, not a limit, since nothing stops a family from having more than ten
members as long as they are contiguous from wherever the scan has reached.

## `suffix motion list`

**Contract** — the same, but the suffixes come from a fixed, ordered table of names supplied
by the declaring code rather than from counting. The result list is exactly as long as the
table, so a missing motion leaves an empty handle *in place* and the index stays meaningful.
This is what lets a creature say "the animation for turning left is entry 2" and have that
be true whether or not entries 0 and 1 loaded.

```text
FUNCTION load_by_suffix(model, base, suffixes : list<text>) -> MotionList
  out = list of (length of suffixes) empty handles
  FOR EACH (i, s) IN suffixes
    out[i] = model.find_cycle(base + s)     # empty handle is a legal outcome
  RETURN out
```

**Notes** — in the original the suffix table is a compile-time template parameter, which is
purely a way of getting a static array of names into the loader without an extra argument.
A rebuild passes the table; nothing depends on it being known at compile time.

## `nested motion collection`

**Contract** — one level of composition: a table of suffixes selects *sub-groups*, and each
sub-group is loaded by one of the two loaders above against the extended base name. This is
how a creature that has four attack variants, each with several random takes, is expressed:
the outer table names the variants, the inner loader collects the takes.

```text
FUNCTION load_collection(model, base, suffixes, inner_loader) -> list<MotionList>
  RETURN [ inner_loader(model, base + s) FOR EACH s IN suffixes ]
```

**Notes** — the recursion stops at one level in practice; nothing forbids more.
