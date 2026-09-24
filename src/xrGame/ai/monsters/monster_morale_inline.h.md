# src/xrGame/ai/monsters/monster_morale_inline.h

> The mode setters, the nudge primitive, and the one question the brains actually ask of morale.

**Needs** — [`monster_morale.h`](monster_morale.h.md)
**Used by** — [`monster_morale.cpp`](monster_morale.cpp.md) · [`monster_morale.h`](monster_morale.h.md)
**Tier floor** — T3: assignments and a comparison

## Purpose

The short half of morale, kept separate only so it could be inlined at every call site. Merge
it into [`monster_morale.cpp`](monster_morale.cpp.md) in a rebuild. Two things here are
load-bearing.

## `set_despondent` / `set_taking_heart` / `set_stable`

**Contract** — set the drift mode. Each is a bare assignment; none touches the value. The mode
is the brain's statement about the situation, and the value is the accumulated consequence, so
setting a mode has no immediate effect at all — only a rate change.

## `nudge`

**Contract** — the private primitive behind the two discrete events: add a signed amount and
clamp into the unit interval.

## `is_despondent`

**Contract** — the question the brains branch on. True when the **mode** is despondent, *or*
when the value has fallen below the authored threshold.

```text
FUNCTION is_despondent() -> bool
  RETURN mode = despondent OR value < despondent_threshold
```

**Notes** — the disjunction is the whole design. A creature counts as broken either because
something declared it broken — the mode, set by the brain — or because it has been ground down
regardless of what the brain thinks. Those are two independent routes to the same behaviour,
and a rebuild that collapses them into one loses the ability to break a creature instantly
without touching its accumulated value.

The consequence is that a creature in despondent mode reports broken even at full morale, and
one in stable mode reports broken while its value is still climbing back through the threshold.
Both are intended.

## `morale`

**Contract** — the raw value, for the brains that want a magnitude rather than the boolean.
